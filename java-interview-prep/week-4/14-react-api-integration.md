# Week 4 — React API Integration Basics (for Backend Devs)

> 📖 **Estimated reading time:** 30 minutes  
> 🎯 **For backend devs:** You won't build React apps, but you should understand what frontend needs from your API  
> ⚠️ **This is NOT a React course** — it's "How your Spring Boot API looks from React's perspective"

---

## 1. How React Consumes REST APIs

### 💻 Typical Pattern

```javascript
// React component — fetches from your Spring Boot backend
import { useEffect, useState } from 'react';

export function PatientList() {
  const [patients, setPatients] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Fetch on component mount
    fetch('/api/patients?page=0&size=20&sort=createdAt,desc')
      .then(response => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      })
      .then(data => {
        setPatients(data.data);  // Your API response enelope
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []); // Empty dependency array = run once on mount

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <table>
      <tbody>
        {patients.map(patient => (
          <tr key={patient.id}>
            <td>{patient.name}</td>
            <td>{patient.email}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### ❓ Interview Q: What problems could your API cause for React frontend?

### ✅ Answer:

**Bad API:**
```json
{
  "id": 1,
  "name": "John Doe",
  "sensitiveField": "PHI_DATA_EXPOSED"  // ❌ Oops, shouldn't send this
}
```

**Good API:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "John Doe"
    // PHI not exposed to frontend
  },
  "error": null
}
```

**Tips for backend devs:**
- Return **consistent status codes** (200, 401, 404, 500) — React checks `response.ok`
- **Return error messages**, not stack traces
- **CORS headers** must be set (or frontend gets `Access-Control-Allow-Origin` error)
- **Consistent response envelope** (data, error, success fields) — makes frontend parsing predictable
- **No sensitive data** exposed (PHI, API keys, DB error details)

---

## 2. Data Fetching Patterns

### Pattern 1: Simple GET

```javascript
// What React expects from: GET /api/patients/1
async function fetchPatient(id) {
  const response = await fetch(`/api/patients/${id}`);
  if (!response.ok) throw new Error(`Patient not found`);
  return response.json();
}
```

**Your Spring Boot:**
```java
@GetMapping("/patients/{id}")
public ResponseEntity<ApiResponse<PatientDTO>> getPatient(@PathVariable Long id) {
    Patient patient = patientRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("Patient not found"));
    return ResponseEntity.ok(new ApiResponse<>(true, PatientDTO.from(patient), null));
}
```

### Pattern 2: POST with Body

```javascript
// What React sends: POST /api/patients
async function createPatient(patientData) {
  const response = await fetch('/api/patients', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(patientData)  // { name, email, ... }
  });
  if (!response.status === 201) throw new Error('Failed to create');
  return response.json();
}
```

**Your Spring Boot should:**
```java
@PostMapping("/patients")
public ResponseEntity<ApiResponse<PatientDTO>> create(@Valid @RequestBody CreatePatientRequest req) {
    // Validate, save, return 201
    Patient patient = patientRepository.save(new Patient(req.name(), req.email()));
    return ResponseEntity
        .status(HttpStatus.CREATED)  // 201, not 200
        .header("Location", "/api/patients/" + patient.getId())  // where to find it
        .body(new ApiResponse<>(true, PatientDTO.from(patient), null));
}
```

### Pattern 3: Error Handling

```javascript
// What React expects on error
fetch('/api/patients/999')
  .then(response => {
    if (response.status === 404) {
      // Your API returned 404, good
      return response.json().then(errorObj => {
        throw new Error(errorObj.error);  // User-friendly message
      });
    }
    if (response.status === 500) {
      // Server error — React shows generic message
      throw new Error('Internal server error');
    }
  })
  .catch(err => {
    console.error(err); // Show to user
  });
```

**Your Spring Boot should return:**
```java
@ExceptionHandler(NotFoundException.class)
public ResponseEntity<ApiResponse<Void>> handleNotFound(NotFoundException ex, HttpServletRequest request) {
    return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body(new ApiResponse<>(false, null, "Patient not found"));
}
```

---

## 3. Common Frontend-Backend Mismatches

### ❌ Problem 1: Missing CORS Header

React app at `http://localhost:3000` tries to fetch from Spring Boot at `http://localhost:8080`

```javascript
fetch('http://localhost:8080/api/patients')
  // Error: Access to XMLHttpRequest has been blocked by CORS policy
```

**Fix in Spring Boot:**
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")  // React dev server
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true);
    }
}
```

### ❌ Problem 2: No Status Code on Create

React expects **201 Created**, gets **200 OK** instead.

```javascript
const response = await fetch('/api/patients', { method: 'POST', ... });
if (response.status !== 201) {
  console.error('Expected 201, got', response.status);
}
```

**Always return correct HTTP status:**
```java
return ResponseEntity.status(HttpStatus.CREATED).body(...);  // 201
// not
return ResponseEntity.ok(...);  // 200 − wrong for POST
```

### ❌ Problem 3: Inconsistent Field Names

React code:
```javascript
const patientName = patient.patientName;  // ← expects this
```

Your API returns:
```json
{ "name": "John" }  // ← returns this instead
```

**Use `@JsonProperty` to match frontend expectations:**
```java
@Entity
public class Patient {
  @JsonProperty("patientName")  // Send as patientName to frontend
  private String name;
}
```

---

## 4. Authentication from React Perspective

### Pattern: JWT in Authorization Header

```javascript
// React stores JWT after login
const token = localStorage.getItem('token');

// Sends it in Authorization header on every request
const response = await fetch('/api/patients', {
  headers: {
    'Authorization': `Bearer ${token}`  // Your Spring Security expects this
  }
});
```

**Your Spring Boot doesn't need to do anything special** — `HttpSecurity` config already reads this header and validates the JWT.

### Common Gotcha: Token Expiry

React receives 401 response (token expired). Should:
1. Redirect to login page
2. Or refresh token if you have refresh token endpoint

**Good practice:** Include refresh token in API response
```json
{
  "success": true,
  "data": { "token": "...", "refreshToken": "..." },
  "expiresIn": 3600
}
```

---

## 5. Pagination from Frontend

### What React Sends

```javascript
// User clicks "page 2"
fetch('/api/patients?page=1&size=20&sort=createdAt,desc')
```

### What Your API Should Return

```java
@GetMapping("/patients")
public ResponseEntity<ApiResponse<Page<PatientDTO>>> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "createdAt,desc") String sort,
    // Spring converts to Pageable
) {
    Page<Patient> patients = patientRepository.findAll(
        PageRequest.of(page, size, Sort.by(..))
    );
    Page<PatientDTO> dtos = patients.map(PatientDTO::from);
    
    return ResponseEntity.ok(new ApiResponse<>(true, dtos, null));
}
```

**React expects in response:**
```json
{
  "success": true,
  "data": {
    "content": [...],        // page.getContent()
    "totalElements": 1000,   // page.getTotalElements()
    "totalPages": 50,        // page.getTotalPages()
    "currentPage": 1,        // page.getNumber()
    "pageSize": 20           // page.getSize()
  }
}
```

Using Spring's `Page` serialization — you get this automatically.

---

## 6. Interview Answer: "How Would You Build a React Form?"

When interviewer asks: *"Show me how you'd build a React form that POSTs patient data to your API"*

**Your answer (as backend dev):**

```
1. Form component with useState hooks for each field
2. onChange handlers to update state
3. On submit, fetch POST /api/patients with body = state object
4. Check response.status === 201
5. If success, redirect to patient detail page
6. If 400 (validation error), show error messages from response.error
7. If 500, show generic error
8. Add loading spinner while POST in progress (setLoading state)
9. Validate client-side (HTML5) first, then trust server validation
```

**Example (minimal):**
```javascript
export function PatientForm() {
  const [formData, setFormData] = useState({ name: '', email: '' });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      const response = await fetch('/api/patients', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData)
      });
      if (response.status !== 201) {
        const errorBody = await response.json();
        setError(errorBody.error || 'Unknown error');
        return;
      }
      // Success — redirect or show message
      alert('Patient created!');
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={formData.name} 
        onChange={e => setFormData({...formData, name: e.target.value})} />
      <input name="email" value={formData.email} 
        onChange={e => setFormData({...formData, email: e.target.value})} />
      <button disabled={loading}>{loading ? 'Creating...' : 'Create'}</button>
      {error && <div style={{color: 'red'}}>{error}</div>}
    </form>
  );
}
```

This shows you understand:
- React hooks (useState)
- Async/await fetch
- Error handling
- HTTP status codes
- Your API contract

**Interviewer is happy → you answer the spirit of the question without claiming React expertise.**

---

## 7. Interview Q&A — React Integration

### Q1: "If the frontend keeps getting 401 on a valid patient GET request, what could be wrong?"

**A:**
1. **JWT expired** — frontend should refresh
2. **JWT token not being sent** — check Authorization header in network tab
3. **CORS misconfiguration** — preflight request fails
4. **Security filter chain issue** — check Spring Security config, maybe `/api/**` not in `permitAll()`

Test with `curl`:
```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/patients
```

### Q2: "How do you handle pagination efficiently on the frontend for a large result set?"

**A:**
- Don't load all 10000 records at once
- Use backend pagination: `?page=0&size=20`
- Lazy-load more pages as user scrolls (infinite scroll) or clicks next
- Cache downloaded pages in React useState to avoid re-fetching

### Q3: "Patient form submits successfully but React doesn't redirect. Why?"

**A:**
- Check API returned 201 (not 200)
- Check frontend code actually calls `redirect()` or `navigate()` after check
- Network tab: confirm 201 response received

---

## Checklist for Interview

- [ ] Know how React sends Authorization header (Bearer token)
- [ ] Understand HTTP status codes from frontend perspective (201 for POST, not 200)
- [ ] Know CORS is a security feature, not a bug
- [ ] Can explain pagination (page, size, sort parameters)
- [ ] Can write simple fetch + error handling
- [ ] Know React hooks basics (useState, useEffect)

---

*You're a backend dev. You don't need React mastery. But understanding how frontend consumes your API makes you 10x more valuable in interviews. 💡*

