```mermaid
sequenceDiagram
    actor Member
    participant SKJ Booking form
    participant Storage Service
    participant SKJ Booking Service
    participant Member Service
    participant Booking DB
    actor Dealer
    participant SKJ Booking CMS
    participant SKJ Journey System

    Member->>+SKJ Booking form: Submits Booking Details & Uploads Receipt
    activate SKJ Booking form

    %% --- Parallel actions: upload receipt and create booking ---
    SKJ Booking form->>+Storage Service: Uploads receipt file
    Storage Service-->>-SKJ Booking form: Returns secure receiptUrl

    SKJ Booking form->>+SKJ Booking Service: POST /api/bookings (bookingDetails, receiptUrl)
    deactivate SKJ Booking form

    activate SKJ Booking Service
    SKJ Booking Service->>+Member Service: validateToken(authToken)
    Member Service-->>-SKJ Booking Service: Returns 200 OK (Member authorized)

    SKJ Booking Service->>+Booking DB: INSERT booking (status: 'Awaiting Approval')
    Booking DB-->>-SKJ Booking Service: Returns bookingId
    deactivate SKJ Booking Service

    %% --- Dealer Approval Sub-flow ---
    Dealer->>+SKJ Booking CMS: Views Pending Bookings
    SKJ Booking CMS->>+SKJ Booking Service: GET /api/bookings?status=AwaitingApproval
    activate SKJ Booking Service
    SKJ Booking Service->>+Booking DB: SELECT * FROM bookings WHERE status='AwaitingApproval'
    Booking DB-->>-SKJ Booking Service: Returns list of pending bookings
    SKJ Booking Service-->>-SKJ Booking CMS: Responds with pending bookings
    deactivate SKJ Booking Service

    SKJ Booking CMS->>Dealer: Displays booking details and receipt for review
    Dealer->>+SKJ Booking CMS: Approves Booking
    SKJ Booking CMS->>+SKJ Booking Service: POST /api/bookings/{id}/approve
    activate SKJ Booking Service

    SKJ Booking Service->>+Booking DB: UPDATE booking SET status='Approved'
    Booking DB-->>-SKJ Booking Service: Success

    %% --- Handoff to Journey System ---
    Note over SKJ Booking Service, SKJ Journey System: Handoff via Event Publication / API Call
    SKJ Booking Service->>+SKJ Journey System: Publishes BookingApproved Event (bookingDetails)
    activate SKJ Journey System
    SKJ Journey System-->>-SKJ Booking Service: Acknowledges Event (202 Accepted)
    deactivate SKJ Journey System

    SKJ Booking Service-->>-SKJ Booking CMS: Returns 200 OK
    deactivate SKJ Booking Service
    SKJ Booking CMS-->>-Dealer: Displays "Booking Approved & Transferred"

