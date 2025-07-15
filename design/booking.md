summary of the roles for each system and component
| System/Component | Primary Role & Responsibilities |
|---|---|
| Member Service | Manages member profiles (CRUD), handles user authentication (including social login via LINE Platform), validates session tokens, and manages authorization. |
| SKJ Booking Service | Orchestrates the entire car booking process from initiation until dealer approval. It validates user requests and coordinates with the Member, PII, and Consent services. |
| PII Data Service | Acts as a secure vault for all Personally Identifiable Information (PII) like contact details and payment receipts. Enforces strict, token-based access controls for all data requests. |
| Consent Service | Manages and provides an audit trail for user consent. It is called whenever an action requires explicit permission from the member to process their data. |
| SKJ Journey System | Manages the post-approval customer lifecycle. It receives approved booking details from the Booking Service and handles subsequent steps like quotation, payment, and delivery. |
| SKJ Booking form (Frontend) | The user interface for members to log in, manage their profile, and create car bookings. It renders the form and communicates with the various backend APIs. |
| SKJ Booking CMS | The internal admin portal for dealership staff. It allows authorized dealers to view pending bookings, review customer-provided details, and approve transactions. |
| LINE Platform | An external identity provider responsible for authenticating users via their LINE account during the social login process. |

```mermaid
sequenceDiagram
    actor Member
    participant SKJ Booking form
    participant LINE Platform
    participant Member Service
    participant Consent Service
    participant PII Data Service
    actor Dealer
    participant SKJ Booking CMS
    participant SKJ Journey System

    %% --- Member Login via LINE ---
    Member->>+SKJ Booking form: Clicks "Login with LINE"
    SKJ Booking form-->>Member: Redirects to LINE Login consent screen
    Member->>+LINE Platform: Enters credentials & approves permissions
    LINE Platform-->>Member: Redirects back to SKJ Booking form with authorization code
    Member->>+SKJ Booking form: Accesses redirect URI with auth code
    SKJ Booking form->>+Member Service: POST /api/login/line (authCode)
    activate Member Service
    Member Service->>+LINE Platform: Exchange auth code for tokens
    LINE Platform-->>-Member Service: Returns access_token, id_token
    Note right of Member Service: Validates ID token, finds or creates member, generates session JWT.
    Member Service-->>-SKJ Booking form: Returns session JWT
    deactivate Member Service
    SKJ Booking form-->>-Member: Displays "Login Successful"

    %% --- Booking Submission by Logged-in Member ---
    Member->>+SKJ Booking form: Fills form, provides PII details, attaches receipt
    SKJ Booking form->>+SKJ Booking Service: POST /api/bookings (bookingDetails, receiptFile, session JWT)
    deactivate SKJ Booking form
    
    activate SKJ Booking Service
    
    %% --- Validation and Secure Data Handling ---
    SKJ Booking Service->>+Member Service: validateToken(session JWT)
    Member Service-->>-SKJ Booking Service: Returns 200 OK (Member authorized)
    
    SKJ Booking Service->>+Consent Service: recordConsent(memberId, 'booking_pii')
    Consent Service-->>-SKJ Booking Service: Returns Success
    
    Note over SKJ Booking Service, PII Data Service: Accessing PII Service with Member's Token
    SKJ Booking Service->>+PII Data Service: storeData(contactDetails, receiptFile, session JWT)
    activate PII Data Service
    Note left of PII Data Service: Validates member's JWT before storing.
    PII Data Service-->>-SKJ Booking Service: Returns secureReferenceIds
    deactivate PII Data Service
    
    Note right of SKJ Booking Service: Creates booking internally with status 'Awaiting Approval', linking PII references.
    deactivate SKJ Booking Service

    %% --- Dealer Approval Sub-flow ---
    Dealer->>+SKJ Booking CMS: Views Pending Bookings
    SKJ Booking CMS->>+SKJ Booking Service: GET /api/bookings?status=AwaitingApproval
    
    activate SKJ Booking Service
    Note over SKJ Booking Service, PII Data Service: Service-to-Service auth for privileged access
    SKJ Booking Service->>+PII Data Service: getDataForDisplay(pii_references, serviceAuthToken)
    activate PII Data Service
    Note left of PII Data Service: Authorizes trusted service call for dealer review.
    PII Data Service-->>-SKJ Booking Service: Returns viewable PII data
    deactivate PII Data Service
    
    SKJ Booking Service-->>-SKJ Booking CMS: Responds with pending booking details
    deactivate SKJ Booking Service
    
    SKJ Booking CMS->>Dealer: Displays booking details and receipt for review
    Dealer->>+SKJ Booking CMS: Approves Booking
    SKJ Booking CMS->>+SKJ Booking Service: POST /api/bookings/{id}/approve
    
    activate SKJ Booking Service
    Note right of SKJ Booking Service: Updates booking status to 'Approved' internally.
    
    %% --- Handoff to Journey System via API---
    Note over SKJ Booking Service, SKJ Journey System: Handoff via Synchronous API Call
    SKJ Booking Service->>+SKJ Journey System: POST /api/journeys (bookingDetails, pii_references)
    activate SKJ Journey System
    Note right of SKJ Journey System: Creates new journey record internally.
    SKJ Journey System-->>-SKJ Booking Service: Returns 201 Created (journeyId)
    deactivate SKJ Journey System
    
    SKJ Booking Service-->>-SKJ Booking CMS: Returns 200 OK
    deactivate SKJ Booking Service
    SKJ Booking CMS-->>-Dealer: Displays "Booking Approved & Transferred"
```

