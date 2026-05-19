# Week 4 — Healthcare & DICOM Basics (for Backend Devs)

> 📖 **Estimated reading time:** 20 minutes  
> 🎯 **What to answer if asked:**  
> - "What is DICOM and why does healthcare use it?"
> - "How would you store/retrieve DICOM images?"
> - "What's dcm4che?"  
> - "How is DICOM metadata relevant to your API design?"

---

## 1. DICOM Deep Dive

### What is DICOM?

**DICOM** = **Digital Imaging and Communications in Medicine**

A **standard** (not a format like JPG) that specifies:
- How to **encode** medical images (X-ray, MRI, CT, Ultrasound, etc.)
- How to **transmit** images between hospital systems
- How to **store** images and metadata
- How systems **communicate** (query/retrieve images)

Think of it like **SMTP for email** — a protocol that lets hospitals reliably exchange imaging data.

### DICOM File Format

```
DICOM File (.dcm) structure:

[File Header (128 bytes of padding)]
[DICM magic number (4 bytes)]
[Meta Information Group Length]
[Meta Information Elements]
|
├─ Transfer Syntax UID (defines encoding: explicit/implicit, compression)
├─ Media Storage SOP Class UID (what type: CR, CT, MR, etc.)
└─ Media Storage SOP Instance UID (unique image ID)
|
[Dataset]
├─ Patient Name: "John Doe"
├─ Patient ID: "12345"
├─ Study Date: "20240519"
├─ Modality: "CT"  (Computed Tomography)
├─ Image Data: [raw pixel array, e.g., 512x512 bytes]
└─ Thousands of other tags...
```

### Interview Q: "What's in a DICOM file?"

**A:**
- **Header:** File preamble + magic bytes
- **Meta info:** How the file is encoded (Transfer Syntax)
- **Dataset:** Patient demographics + image metadata + actual pixel data
  - Thousands of **tags** (group, element) with structured data
  - Example: `(0010, 0010)` = Patient Name
  - Example: `(7FE0, 0010)` = Pixel Data (the actual image)

**Key point:** DICOM isn't just image data. It's **image + metadata + patient info bundled together**. That's why it's 50–200 MB per image (uncompressed).

---

## 2. Why Healthcare Standardized on DICOM

### Problem Before DICOM
- Hospital A scans patient → proprietary format from Siemens machine
- Patient visits Hospital B → can't read Siemens format
- Doctor has to re-scan (expensive, radiation exposure)

### Solution: DICOM Standard (1985+)
- **All manufacturers** (Siemens, GE, Philips) agreed on DICOM format
- Images portable across hospitals
- Metadata standardized (patient ID, modality, etc.)

---

## 3. Modalities & Use Cases

| Modality | Full Name | Image Type | File Size |
|----------|-----------|-----------|-----------|
| **CR** | Computed Radiography | X-ray | 10–50 MB |
| **CT** | Computed Tomography | 3D volumetric scans | 100–500 MB per series |
| **MR** | Magnetic Resonance | MRI scans | 50–300 MB per series |
| **US** | Ultrasound | Real-time imaging | 5–50 MB |
| **PT** | Positron Emission | PET scans | 100–200 MB |
| **OT** | Other | Screenshots, diagrams | 1–10 MB |

**Series vs Instance:**
- **Study** = all images from one patient visit (can have multiple modalities)
- **Series** = one type of image from one study (e.g., "CT Chest with Contrast — 200 slices")
- **Instance** = individual image/slice

---

## 4. DICOM in Spring Boot Backend

### Reading DICOM Files

**Option 1: Use dcm4che library (Java)**

```java
// Maven dependency
<dependency>
    <groupId>org.dcm4che</groupId>
    <artifactId>dcm4che-core</artifactId>
    <version>5.29.2</version>
</dependency>

// Code
import org.dcm4che3.data.Attributes;
import org.dcm4che3.io.DicomInputStream;

@Service
public class DicomService {
    
    public DicomMetadata readDicomFile(Path filePath) throws IOException {
        try (DicomInputStream dis = new DicomInputStream(filePath.toFile())) {
            Attributes attrs = dis.readDataset(-1, -1);  // read all tags
            
            String patientName = attrs.getString(Tag.PatientName, "Unknown");
            String patientID = attrs.getString(Tag.PatientID);
            String modality = attrs.getString(Tag.Modality);  // "CT", "MR", etc.
            String sopInstanceUID = attrs.getString(Tag.SOPInstanceUID);
            
            byte[] pixelData = attrs.getBytes(Tag.PixelData);
            int width = attrs.getInt(Tag.Columns, 0);
            int height = attrs.getInt(Tag.Rows, 0);
            
            return new DicomMetadata(patientName, patientID, modality, sopInstanceUID, width, height);
        }
    }
}

record DicomMetadata(String patientName, String patientId, String modality, 
                     String instanceUid, int width, int height) { }
```

### Storing DICOM Files

**NOT in database.** Store in:
1. **File system** (if single server)
2. **AWS S3** (if cloud, distributed)
3. **DICOM Server** (specialized: dcm4chee, Orthanc)

**Pattern:**
```java
@Service
public class DicomStorageService {
    
    @Value("${dicom.storage.path:./dicom-files}")
    private String storagePath;
    
    public void storeDicomFile(MultipartFile file, String patientId, String studyInstanceUid) 
            throws IOException {
        
        // Directory structure: /dicom-files/patient-id/study-uid/
        Path directory = Paths.get(storagePath, patientId, studyInstanceUid);
        Files.createDirectories(directory);
        
        // Generate unique filename
        String filename = UUID.randomUUID() + ".dcm";
        Path filePath = directory.resolve(filename);
        
        // Save file
        Files.write(filePath, file.getBytes());
        
        // Store metadata in DB (for indexing)
        DicomFileMetadata metadata = new DicomFileMetadata(
            patientId, studyInstanceUid, filename, filePath.toString(), 
            file.getSize(), LocalDateTime.now()
        );
        dicomMetadataRepository.save(metadata);
    }
}
```

### Retrieving DICOM Images

**REST endpoint:**
```java
@GetMapping("/api/dicom/{patientId}/studies/{studyUid}")
public ResponseEntity<byte[]> getDicomImage(
        @PathVariable String patientId,
        @PathVariable String studyUid) {
    
    // Find metadata in DB
    DicomFileMetadata metadata = dicomRepository.findByPatientIdAndStudyUid(patientId, studyUid)
        .orElseThrow(() -> new NotFoundException("DICOM not found"));
    
    // Check authorization (HIPAA requirement!)
    if (!canAccessPatientData(Authentication.getPrincipal(), patientId)) {
        throw new ForbiddenException("Access denied to this patient's data");
    }
    
    // Stream file
    byte[] fileContent = Files.readAllBytes(Paths.get(metadata.getFilePath()));
    
    return ResponseEntity.ok()
        .header("Content-Type", "application/dicom")
        .header("Content-Disposition", "inline; filename=" + metadata.getFilename())
        .body(fileContent);
}
```

---

## 5. DICOM Query/Retrieve (C-FIND, C-MOVE)

DICOM Network Protocol (not HTTP, but **socket-based**):

### C-FIND Request (Query)
```
Client → DICOM Server: "Find all CT images for patient ID 12345"
DICOM Server → Client: List of matching DICOM studies
```

**In Spring Backend:**
```java
// Usually you DON'T implement DICOM protocol yourself
// Instead, use a DICOM server (like Orthanc) and query its REST API

RestTemplate restTemplate = new RestTemplate();
String orthanResponse = restTemplate.getForObject(
    "http://dicom-server:8042/api/patients/12345/studies",
    String.class
);
```

### Why Spring Backend Might Not Touch DICOM Protocol

Modern healthcare uses **DICOM servers** (Orthanc, dcm4chee) that:
- Accept DICOM files
- Index them
- Provide REST API for queries

Your Spring Backend calls their REST API:
```
GET http://dicom-server:8042/api/studies?PatientName=John*
```

---

## 6. Healthcare & DICOM Context for AdvaHealth

### What AdvaHealth Likely Does

From JD: *"you build APIs that integrate with EHRs, imaging systems"*

This means:
1. **Receive DICOM images** from imaging modalities (CT machines, X-ray machines) or DICOM servers
2. **Parse metadata** (patient ID, study date, modality)
3. **Store** in cloud (S3) + metadata in PostgreSQL
4. **Serve** to clinicians via REST API
5. **Integrate** with EHR (Epic, Cerner) via HL7/FHIR messages

### Your Role as Backend Dev

**You don't need to:**
- Understand DICOM pixel data compression algorithms
- Write DICOM server software
- Implement C-STORE protocol

**You DO need to:**
- Know what DICOM is
- Understand metadata fields (patient ID, study UID)
- Secure PHI (HIPAA requirements)
- Design APIs that handle large files (100+ MB DICOM studies)
- Index and query DICOM metadata efficiently

---

## 7. Interview Answers

### Q: "What's DICOM and why does healthcare use it?"

**A:**
DICOM is a **standard format** for medical images — lets different vendors (Siemens, GE, etc.) exchange patient images. Files contain image data + metadata (patient name, modality, etc.). Without it, hospitals couldn't share images → patients would need re-scans.

### Q: "How would you store DICOM images in AdvaHealth's SaaS platform?"

**A:**
1. Receive DICOM file (via HTTP upload or DICOM protocol integrations)
2. Parse metadata with dcm4che library (Java)
3. Store binary file in S3 (with client-side encryption for HIPAA)
4. Store metadata in PostgreSQL indexed by patient ID + study date
5. Serve via REST: GET /api/patients/{id}/studies returns list with presigned S3 URLs
6. Log all access for audit trail (HIPAA Accounting of Disclosures)

### Q: "What security concerns exist with DICOM data?"

**A:**
- **HIPAA mandates:** Encryption at rest + in transit (TLS)
- **Access control:** Only authorized clinicians can view patient images
- **Audit logging:** Track who accessed which images (Accounting of Disclosures)
- **De-identification:** Remove PHI from teaching/research images
- **Data residency:** Some healthcare laws require data to stay in-country

### Q: "What's the difference between HL7 and FHIR?"

**A:**
- **HL7 v2:** Old messaging standard (1990s), pipe-delimited text, hospitals use it for EHR integration
- **FHIR:** Modern REST-based standard (2010s), JSON/XML, RESTful endpoints

Example FHIR endpoint (that your API might integrate with):
```
GET /fhir/Patient/12345  → returns patient demographics
GET /fhir/DiagnosticReport?patient=12345  → returns imaging reports
```

---

## 8. Minimal DICOM Knowledge for Interview

| Topic | Remember This |
|-------|---|
| **What is DICOM** | Standard format for medical images (not JPG) |
| **File size** | Large: 50–500 MB per study (hundreds of slices) |
| **Metadata** | Patient ID, Study Date, Modality (CT, MR, etc.), pixel dimensions |
| **Storage** | Don't store in DB; use S3 or DICOM server |
| **Why it matters** | Lets hospitals share images reliably |
| **Security** | HIPAA requires encryption + access control |
| **Your role** | API to access DICOM metadata/files, not understanding pixel data |

---

## Checklist Before Interview

- [ ] Can define DICOM in one sentence
- [ ] Know what dcm4che is (Java library to read DICOM)
- [ ] Understand modalities: CT, MR, CR, US
- [ ] Can explain why hospitals use DICOM
- [ ] Know HIPAA applies (encryption, audit logging)
- [ ] Can describe a simple flow: upload → parse → store → serve

---

*DICOM is not complex for backend devs. It's just a file format with lots of metadata. The real challenge is healthcare requirements (HIPAA) not DICOM itself.* 💊


