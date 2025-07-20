```mermaid
sequenceDiagram
    participant User
    participant Web Application (Frontend)
    participant CIAM System (Backend)
    participant LINE Platform

    %% Part 1: Initial User Registration with Email and OTP %%
    box rgba(240, 248, 255, 0.5) Part 1: Register with Email & OTP
    User->>Web Application (Frontend): 1. Fills out registration form (Email, Password, etc.)
    Web Application (Frontend)->>CIAM System (Backend): 2. Sends registration details (API call)
    note right of Web Application (Frontend): Initiates the registration process.

    CIAM System (Backend)->>CIAM System (Backend): 3. Creates temporary user record, generates OTP
    CIAM System (Backend)->>User: 4. Sends OTP to user's email

    CIAM System (Backend)-->>Web Application (Frontend): 5. Responds that OTP has been sent
    Web Application (Frontend)-->>User: 6. Prompts user to enter OTP

    User->>Web Application (Frontend): 7. Enters OTP from their email and submits

    Web Application (Frontend)->>CIAM System (Backend): 8. Sends OTP and final details for verification
    
    CIAM System (Backend)->>CIAM System (Backend): 9. Validates OTP
    alt OTP is Correct
        CIAM System (Backend)->>CIAM System (Backend): 10a. Activates user account, securely hashes password
        CIAM System (Backend)-->>Web Application (Frontend): 11a. Returns success, issues session tokens
        Web Application (Frontend)-->>User: 12a. Logs user in, shows success message
    else OTP is Incorrect
        CIAM System (Backend)-->>Web Application (Frontend): 10b. Returns error response
        Web Application (Frontend)-->>User: 11b. Displays "Invalid OTP" error
    end
    end

    %% Part 2: Optional - Linking LINE Account %%
    box rgba(240, 255, 240, 0.5) Part 2: Optional - Link LINE Account (Post-Registration)
    note over User, Web Application (Frontend): User is already logged in and decides to link their LINE account from their profile page.
    
    User->>Web Application (Frontend): 13. Clicks "Connect with LINE"
    Web Application (Frontend)->>CIAM System (Backend): 14. Initiates OpenID Connect (OIDC) flow for account linking

    CIAM System (Backend)->>User: 15. Redirects user's browser to LINE Platform for authentication
    
    User->>LINE Platform: 16. Logs into LINE and grants consent
    
    LINE Platform-->>User: 17. Redirects user back to CIAM System with an authorization code
    
    User->>CIAM System (Backend): 18. Browser delivers the authorization code
    
    CIAM System (Backend)->>LINE Platform: 19. Exchanges authorization code for tokens
    note right of CIAM System (Backend): Gets ID Token containing the user's unique LINE ID.
    
    LINE Platform-->>CIAM System (Backend): 20. Returns ID Token and Access Token
    
    CIAM System (Backend)->>CIAM System (Backend): 21. Validates ID token and extracts LINE user ID
    CIAM System (Backend)->>CIAM System (Backend): 22. Links the LINE ID to the existing user account in its database
    
    CIAM System (Backend)-->>User: 23. Redirects user back to the Web Application
    
    User->>Web Application (Frontend): 24. Browser lands on the specified redirect URI
    Web Application (Frontend)-->>User: 25. Displays "LINE account successfully linked!" message
    end

