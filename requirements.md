# Backend Requirements Specification

## 1. User Authentication System

### Overview
Handles user registration, login, and session management for guests and hosts with JWT-based authentication.

### API Endpoints

#### POST /api/auth/register
**Purpose**: Create new user account

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "securePassword123",
  "firstName": "John",
  "lastName": "Doe",
  "userType": "guest|host",
  "phone": "+1234567890"
}
```

**Response (201)**:
```json
{
  "success": true,
  "user": {
    "id": "usr_123",
    "email": "user@example.com",
    "firstName": "John",
    "userType": "guest"
  },
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### POST /api/auth/login
**Purpose**: Authenticate existing user

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

**Response (200)**:
```json
{
  "success": true,
  "user": {
    "id": "usr_123",
    "email": "user@example.com",
    "firstName": "John"
  },
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expiresAt": "2025-07-25T10:30:00Z"
}
```

#### POST /api/auth/oauth/google
**Purpose**: OAuth authentication via Google

**Request Body**:
```json
{
  "idToken": "google_oauth_token",
  "userType": "guest|host"
}
```

### Validation Rules
- Email must be valid format and unique
- Password minimum 8 characters with one uppercase, lowercase, number
- Phone number must match international format
- userType limited to 'guest' or 'host'
- OAuth tokens verified against provider

### Security Requirements
- Passwords hashed using bcrypt (cost factor 12)
- JWT tokens expire after 24 hours
- Rate limiting: 5 failed attempts per IP per 15 minutes
- HTTPS required for all auth endpoints
- Input sanitization to prevent injection attacks

### Performance Criteria
- Registration response time < 500ms
- Login response time < 200ms
- Support 1000 concurrent login requests
- Database query optimization with indexed email lookups

---

## 2. Property Management System

### Overview
Allows hosts to create, update, and manage property listings with comprehensive details and availability.

### API Endpoints

#### POST /api/properties
**Purpose**: Create new property listing
**Auth**: Required (Host only)

**Request Body**:
```json
{
  "title": "Cozy Downtown Apartment",
  "description": "Beautiful 2BR apartment...",
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipCode": "94102",
    "country": "US"
  },
  "pricePerNight": 150.00,
  "maxGuests": 4,
  "bedrooms": 2,
  "bathrooms": 1,
  "amenities": ["wifi", "kitchen", "parking"],
  "images": ["image1.jpg", "image2.jpg"],
  "checkInTime": "15:00",
  "checkOutTime": "11:00"
}
```

**Response (201)**:
```json
{
  "success": true,
  "property": {
    "id": "prop_456",
    "title": "Cozy Downtown Apartment",
    "status": "draft",
    "createdAt": "2025-07-24T10:00:00Z"
  }
}
```

#### GET /api/properties/{propertyId}
**Purpose**: Retrieve property details

**Response (200)**:
```json
{
  "success": true,
  "property": {
    "id": "prop_456",
    "title": "Cozy Downtown Apartment",
    "description": "Beautiful 2BR apartment...",
    "pricePerNight": 150.00,
    "rating": 4.8,
    "reviewCount": 24,
    "availability": {
      "2025-07-25": true,
      "2025-07-26": false
    }
  }
}
```

#### PUT /api/properties/{propertyId}
**Purpose**: Update property details
**Auth**: Required (Property owner only)

#### DELETE /api/properties/{propertyId}
**Purpose**: Delete property listing
**Auth**: Required (Property owner only)

### Validation Rules
- Title: 10-100 characters, required
- Description: 50-2000 characters, required
- Price: Positive decimal, max 2 decimal places
- Max guests: Integer between 1-16
- Address fields required except apartment number
- Amenities from predefined list only
- Images: Max 10 files, each under 5MB
- Check-in/out times in HH:MM format

### Business Rules
- Only property owners can modify their listings
- Properties must be approved before going live
- Minimum 24-hour gap between check-in and check-out times
- Location coordinates auto-generated from address
- Image optimization and CDN upload automated

### Performance Criteria
- Property creation < 2 seconds (including image upload)
- Property search queries < 300ms
- Support 500 concurrent property updates
- Image processing queue for background optimization

---

## 3. Booking System

### Overview
Manages the complete booking lifecycle from availability checking to confirmation with payment integration.

### API Endpoints

#### POST /api/bookings/availability
**Purpose**: Check property availability for dates

**Request Body**:
```json
{
  "propertyId": "prop_456",
  "checkIn": "2025-08-15",
  "checkOut": "2025-08-18",
  "guests": 2
}
```

**Response (200)**:
```json
{
  "success": true,
  "available": true,
  "pricing": {
    "basePrice": 450.00,
    "serviceFee": 45.00,
    "taxes": 36.00,
    "total": 531.00
  }
}
```

#### POST /api/bookings
**Purpose**: Create new booking
**Auth**: Required (Guest only)

**Request Body**:
```json
{
  "propertyId": "prop_456",
  "checkIn": "2025-08-15",
  "checkOut": "2025-08-18",
  "guests": 2,
  "guestDetails": {
    "firstName": "Jane",
    "lastName": "Smith",
    "phone": "+1234567890"
  },
  "specialRequests": "Early check-in if possible"
}
```

**Response (201)**:
```json
{
  "success": true,
  "booking": {
    "id": "book_789",
    "status": "pending_payment",
    "paymentIntent": "pi_stripe_123",
    "expiresAt": "2025-07-24T10:15:00Z",
    "total": 531.00
  }
}
```

#### GET /api/bookings/{bookingId}
**Purpose**: Retrieve booking details
**Auth**: Required (Guest/Host/Admin only)

#### PUT /api/bookings/{bookingId}/cancel
**Purpose**: Cancel existing booking
**Auth**: Required (Guest/Host only)

### Validation Rules
- Check-in date must be future date
- Check-out date must be after check-in
- Guest count must not exceed property capacity
- Booking window: Minimum 1 night, maximum 28 nights
- Guest details required for bookings over $500
- Special requests limited to 500 characters

### Business Logic
- 15-minute payment window after booking creation
- Automatic availability lock during payment process
- Cancellation policies enforced based on timing
- Host notification within 5 minutes of confirmed booking
- Double booking prevention through database constraints

### Performance Criteria
- Availability check < 100ms
- Booking creation < 500ms
- Payment processing < 3 seconds
- Support 200 concurrent booking requests
- Database transactions with ACID compliance

### Integration Requirements
- Stripe/PayPal payment processing
- Email notifications via SendGrid
- Calendar sync for availability updates
- Audit logging for all booking state changes
