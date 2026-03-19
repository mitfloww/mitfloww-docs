# Software Requirements Specification (SRS)
## FileWall – Secure File Delivery Platform

---

# 1. Introduction

## 1.1 Purpose
This document describes the functional and non-functional requirements for **FileWall**, a platform that enables creators to securely share files with clients and ensure access is granted only after successful payment.

## 1.2 Scope
FileWall allows:

- Creators to upload files
- Generate preview versions
- Share secure links
- Lock original files behind payment
- Automatically unlock files after payment confirmation
- Collect client reviews and ratings
- Creator dashboard
- Generate shareable testimonials

The system focuses on **secure delivery, payment assurance, controlled access, and creator reputation building**.

## 1.3 Definitions

| Term | Meaning |
|------|--------|
| Creator | User uploading and selling files |
| Client | User purchasing access |
| Preview | Limited version of file |
| Original | Full file |
| Unlock | Granting access after payment |
| Signed URL | Temporary secure download link |
| Creator dashboard | Track payments and earnings |
| Testimonial | Public representation of client feedback |

---

# 2. Overall Description

## 2.1 Product Perspective

FileWall is a web-based application built using:

- Frontend + Backend: Next.js
- DB: Postgres
- Storage: Object storage (e.g., R2/S3)
- Payment Integration: External provider (e.g., Razorpay/Stripe)

## 2.2 Product Functions

- File upload and storage
- Preview generation
- Secure link sharing
- Payment processing
- Access control
- Download authorization
- Review and rating system
- Creator dashboard
- Testimonial generation

## 2.3 User Classes

### Creator
- Upload files
- Set pricing
- Share links
- Track payments
- View client reviews
- View valuable client, monthly earnings
- Generate testimonials

### Client
- View preview
- Make payment
- Download file
- Provide rating and optional review

---

# 3. System Features

## 3.1 File Upload

### Description
Allows creators to upload files to the system.

### Functional Requirements
- Upload file
- Validate file type/size
- Store securely
- Generate unique ID

---

## 3.2 Preview Generation

### Description
Generate a limited-access version of the file.

### Functional Requirements
- Create low-quality or watermarked version
- Store preview separately
- Link preview to original

---

## 3.3 File Sharing

### Description
Generate shareable link for clients.

### Functional Requirements
- Unique file link
- Public preview access
- Secure identifier

---

## 3.4 Payment Processing

### Description
Handle payments before unlocking files.

### Functional Requirements
- Integrate payment gateway
- Verify payment via webhook
- Handle success/failure

---

## 3.5 File Unlocking

### Description
Grant access to original file after payment.

### Functional Requirements
- Validate payment
- Generate signed download URL
- Expire link after time

---

## 3.6 Download Management

### Description
Allow secure download.

### Functional Requirements
- Temporary access
- Limit download attempts
- Prevent unauthorized access

---

## 3.7 Authentication (Optional Phase)

### Description
User account system.

### Functional Requirements
- Login/Register
- Session management
- Role-based access

---

## 3.8 Review and Rating System

### Description
Allows clients to provide feedback after accessing the file.

### Functional Requirements
- Prompt user for review after download
- Star rating is mandatory
- Text review is optional
- Prevent multiple reviews per transaction
- Store review linked to file and creator
- Allow creators to view reviews in dashboard

---

## 3.9 Creator Dashboard

### Description
Allows creators to view their monthly earnings, most valuable client, payments reports etc.

### Functional Requirements
- Display current month earning
- Display most valuable client name and details
- Display graph to compare weekly/monthly/yearly earnings
- Dislay payment history
- Display pending payments
- Display expired file sharing links
- Display file expiry date and revisions left.
- Display current month usage of FileWall
    - Number of files uploaded per month
    - Storage left
    - Manage files(Download and Delete)

---

## 3.10 Testimonial Generation

### Description
Allows creators to convert reviews into shareable testimonials.

### Functional Requirements
- Display list of reviews
- Select review for testimonial
- Provide predefined templates
- Allow customization:
  - Colors
  - Layout
  - Text styling
- Generate shareable image/card
- Export for social media (LinkedIn, Instagram, etc.)
- Ensure consent before public usage

---

# 4. External Interface Requirements

## 4.1 User Interface

- Web-based UI
- Creator dashboard
- Preview page for clients
- Review submission interface
- Testimonial editor

## 4.2 API Interfaces

- File upload API
- Payment webhook API
- Download API
- Review submission API
- Testimonial generation API

## 4.3 Storage Interface

- Object storage system
- Secure file retrieval

---

# 5. Non-Functional Requirements

## 5.1 Security

- Secure file storage
- Signed URLs
- Access validation
- Payment verification

## 5.2 Performance

- Fast file upload
- Efficient preview loading
- Scalable storage handling

## 5.3 Scalability

- Support increasing users
- Handle large file volumes
- Expandable backend architecture

## 5.4 Reliability

- Ensure file availability
- Accurate payment verification
- Fault tolerance

## 5.5 Usability

- Simple UI
- Minimal steps for file sharing
- Clear user flows

## 5.6 Privacy

- Protect user data
- Do not expose sensitive information in reviews
- Require consent for public testimonials

---

# 6. System Architecture Overview

## 6.1 High-Level Flow

1. Creator uploads file
2. System generates preview
3. Creator shares link
4. Client views preview
5. Client makes payment
6. System verifies payment
7. System unlocks file
8. Client downloads file
9. Client provides rating (mandatory) and optional review

---

# 7. Data Flow

## 7.1 Upload Flow

Creator → Upload API → Storage → Preview Generator → Preview Stored

## 7.2 Payment Flow

Client → Payment Gateway → Webhook → Verification → Unlock File

## 7.3 Download Flow

Client → Request → Validate → Signed URL → Download

## 7.4 Review Flow

Client → Download → Review Prompt → Submit → Store → Dashboard Display

---

# 8. Constraints

- Must use secure storage
- Payment provider dependency
- Internet connection required
- File size limits may apply

---

# 9. Assumptions

- Users have internet access
- Payment gateway is reliable
- Storage system is available
- Files are legal and safe

---

# 10. Future Enhancements

- Advanced analytics dashboard
- Subscription plans
- API access for developers
- Team collaboration
- Mobile application
- Creator hiring
- P2P chat

---

# 11. Appendix

## 11.1 Risks

- Payment fraud
- File piracy
- Unauthorized sharing
- Low-quality or fake reviews
- Storage cost growth

## 11.2 Mitigations

- Signed URLs
- Expiry links
- Watermarked previews
- Rate limiting
- Review validation mechanisms