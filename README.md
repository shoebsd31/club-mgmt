# Pickleball Club Management System - Detailed Requirements

## 1. System Overview

### 1.1 Purpose
A comprehensive mobile and web application system to manage multiple pickleball clubs, including membership management, game voting, payment tracking, and administrative controls.

### 1.2 Scope
- Multi-club support architecture
- Cross-platform mobile applications (iOS & Android)
- Web-based admin panel
- Automated voting and payment systems
- Financial ledger management
- Cloud-based file storage and backup

## 2. User Roles & Permissions

### 2.1 Super Admin
**Responsibilities:**
- Full system access across all clubs
- User role management (promote/demote users)
- Club configuration and setup
- System-wide settings and configurations
- Approve/reject voting changes and cancellations
- Manage court assignments and locations
- Publish weekly polls
- Send membership renewal notices
- Invite new players via integrated membership form

**Permissions:**
- Create, read, update, delete all data
- Access all clubs and locations
- Manage all user accounts
- Configure system-wide settings
- Access all reports and analytics

### 2.2 Ledger Admin
**Responsibilities:**
- Financial transaction management
- Process refunds (requires Super Admin approval)
- Log court fee payments with receipts
- Monitor member account balances
- Generate financial reports
- Manage spending entries with proof documentation

**Permissions:**
- Full access to ledger and financial data
- Create refund requests
- Upload and manage financial documents
- Generate financial reports
- View member payment history

### 2.3 Club Member
**Responsibilities:**
- Participate in weekly voting polls
- Top-up account balance with transfer confirmations
- View personal game history and financial statements
- Update personal profile information

**Permissions:**
- Vote in polls (if not restricted by admin)
- View personal data and history
- Upload payment confirmations
- Request vote changes (subject to admin approval)
- Receive notifications and monthly statements

## 3. Authentication & Security

### 3.1 Login Methods
- Google OAuth 2.0
- Apple Sign-In
- Mobile number verification via OTP

### 3.2 Member Verification Process
1. User attempts registration with mobile number
2. System checks if mobile number exists in approved member database
3. If found, allows registration; if not, displays "Contact admin" message
4. Super Admin can manually add/modify mobile numbers for member verification
5. Two-factor authentication for admin accounts

### 3.3 Access Control
- Role-based permissions (RBAC)
- Club-specific access controls
- Session management with automatic logout
- API rate limiting and security headers

## 4. Database Schema (PostgreSQL)

### 4.1 Core Tables

#### 4.1.1 Clubs
```sql
- club_id (Primary Key)
- club_name
- description
- created_date
- status (active/inactive)
- settings (JSON for club-specific configurations)
```

#### 4.1.2 Users (Admins & Members)
```sql
- user_id (Primary Key)
- email
- mobile_number
- full_name
- role (super_admin, ledger_admin, club_member)
- club_id (Foreign Key)
- google_id
- apple_id
- profile_picture_url
- account_balance (decimal)
- status (active/inactive/suspended)
- created_date
- last_login
```

#### 4.1.3 Locations
```sql
- location_id (Primary Key)
- club_id (Foreign Key)
- location_name
- address
- rental_fee_amount
- rental_fee_period (months)
- game_price_per_person
- number_of_courts
- status (active/inactive)
```

#### 4.1.4 Courts
```sql
- court_id (Primary Key)
- location_id (Foreign Key)
- court_name
- court_number
- status (available/maintenance)
```

#### 4.1.5 Membership
```sql
- membership_id (Primary Key)
- user_id (Foreign Key)
- club_id (Foreign Key)
- membership_fee
- start_date
- end_date
- status (active/expired/pending)
- payment_confirmation_url
```

#### 4.1.6 Polls/Voting
```sql
- poll_id (Primary Key)
- club_id (Foreign Key)
- location_id (Foreign Key)
- poll_title
- game_date
- game_time
- created_date
- published_date
- voting_deadline
- status (draft/published/closed)
- max_players
```

#### 4.1.7 Votes
```sql
- vote_id (Primary Key)
- poll_id (Foreign Key)
- user_id (Foreign Key)
- vote_timestamp
- status (confirmed/cancelled/pending_change)
- change_reason
- admin_approval_status
- admin_approval_by
- admin_approval_timestamp
```

#### 4.1.8 Game Sessions
```sql
- session_id (Primary Key)
- poll_id (Foreign Key)
- court_id (Foreign Key)
- game_date
- game_time
- players (JSON array of user_ids)
- total_amount_collected
- status (scheduled/completed/cancelled)
```

#### 4.1.9 Ledger
```sql
- transaction_id (Primary Key)
- user_id (Foreign Key)
- club_id (Foreign Key)
- transaction_type (topup/game_charge/refund/membership_fee/court_rental)
- amount
- description
- reference_document_url
- created_by (user_id)
- approved_by (user_id)
- approval_status (pending/approved/rejected)
- created_date
- approval_date
```

#### 4.1.10 Configurations
```sql
- config_id (Primary Key)
- club_id (Foreign Key)
- config_key
- config_value (JSON)
- created_date
- updated_date
- updated_by
```

#### 4.1.11 File Storage
```sql
- file_id (Primary Key)
- file_name
- file_path
- file_type
- file_size
- uploaded_by
- upload_date
- metadata (JSON)
- associated_record_type
- associated_record_id
```

#### 4.1.12 Notifications
```sql
- notification_id (Primary Key)
- user_id (Foreign Key)
- title
- message
- type (email/push/sms)
- status (sent/pending/failed)
- sent_date
- read_status
```

## 5. Core Functionalities

### 5.1 Membership Management

#### 5.1.1 Membership Registration
- Integrated membership form within the application
- Collect: Full name, email, mobile number, emergency contact
- Annual membership fee: €30 (configurable per club)
- Payment confirmation upload functionality
- Admin approval workflow

#### 5.1.2 Member Verification
- Mobile number-based verification system
- Admin can add/modify member mobile numbers
- Automatic account creation upon successful verification

#### 5.1.3 Membership Renewal
- Automated renewal reminders (configurable timing)
- Admin-triggered renewal notices
- Renewal payment tracking
- Grace period management

### 5.2 Voting System

#### 5.2.1 Poll Creation & Management
- Automated poll generation based on club configurations
- Location-specific and date-specific polls
- Admin manual publishing control
- Configurable voting deadlines (default: 24 hours before game)

#### 5.2.2 Voting Process
- Members can vote for multiple polls simultaneously
- Real-time vote count display
- Vote modification requests with reason requirement
- Admin approval for vote changes within 24-hour window

#### 5.2.3 Vote Cancellation Rules
- No cancellation allowed within 24 hours (system enforced)
- Emergency cancellations require admin approval
- Valid reason mandatory for last-minute cancellations
- Approved cancellations result in no charge

### 5.3 Payment & Financial Management

#### 5.3.1 Account Balance System
- Individual member account balances
- Top-up functionality with manual confirmation
- Automatic deduction after game completion
- Negative balance notifications (immediate)
- Monthly balance statements

#### 5.3.2 Payment Processing
- Manual transfer confirmation system
- Receipt/proof upload for all transactions
- Pending payment tracking
- Payment history for each member

#### 5.3.3 Game Charging
- Automatic charging post-game based on votes
- Location-specific pricing (L1: €7, L2: €5 per person)
- Failed payment handling
- Charge dispute resolution

#### 5.3.4 Refund Management
- Ledger Admin initiates refunds
- Super Admin approval required
- Refund tracking and audit trail
- Documentation requirements

### 5.4 Court & Location Management

#### 5.4.1 Location Configuration
- Multiple locations per club
- Rental fee tracking (L1: €600/6 months, L2: €1200/12 months)
- Game pricing per location
- Court availability management

#### 5.4.2 Court Assignment
- Admin manual court assignment to players
- Named court system
- Court availability scheduling
- Conflict resolution

### 5.5 Administrative Features

#### 5.5.1 Club Configuration
- Multi-club architecture support
- Club-specific settings and pricing
- Location and court management
- Member management per club

#### 5.5.2 Financial Administration
- Comprehensive ledger system
- Expense tracking with proof documentation
- Court rental payment logging
- Financial report generation

#### 5.5.3 User Management
- Role assignment and modification
- Member suspension/activation
- Voting restrictions per member
- Account balance adjustments

## 6. Mobile Application Features

### 6.1 iOS & Android Applications
- Native performance optimization
- Offline capability for viewing personal data
- Push notification support
- Biometric authentication where available

### 6.2 Core Mobile Features
- Authentication (Google, Apple, Mobile OTP)
- Weekly poll participation
- Account balance monitoring
- Payment confirmation upload
- Personal statistics dashboard
- Notification center

### 6.3 Push Notifications
- New poll availability
- Voting reminders
- Payment due notifications
- Game confirmations
- Account balance alerts
- General club announcements

## 7. Reporting & Analytics

### 7.1 Member Reports
- Monthly personal statements (games played, payments, balance)
- Annual membership summary
- Game participation history
- Payment transaction history

### 7.2 Admin Reports
- Member engagement analytics
- Active player statistics
- Financial summaries (daily/weekly/monthly)
- Court utilization reports
- Payment collection reports
- Outstanding balance reports

### 7.3 Financial Reports
- Club revenue analysis
- Court rental expense tracking
- Member payment trends
- Profit/loss statements
- Budget variance reports

## 8. Technical Requirements

### 8.1 Platform & Architecture
- **Backend**: Node.js/Express.js or Python/Django
- **Database**: PostgreSQL with connection pooling
- **Mobile**: React Native or Flutter for cross-platform
- **Cloud Storage**: AWS S3 or Google Cloud Storage
- **Authentication**: OAuth 2.0, JWT tokens
- **API**: RESTful API design with versioning

### 8.2 Cloud Infrastructure
- **Hosting**: AWS/GCP/Azure cloud platform
- **File Storage**: Secure cloud storage with metadata
- **Backup**: Automated daily database backups
- **CDN**: Content delivery network for file access
- **Monitoring**: Application performance monitoring

### 8.3 Security Requirements
- **Data Encryption**: At rest and in transit
- **API Security**: Rate limiting, input validation
- **File Upload**: Virus scanning, type validation
- **Audit Logging**: All admin actions logged
- **GDPR Compliance**: Data privacy controls

### 8.4 Performance Requirements
- **Response Time**: <2 seconds for API calls
- **Availability**: 99.9% uptime SLA
- **Scalability**: Support for 1000+ concurrent users
- **Mobile Performance**: <3 seconds app launch time

## 9. Integration Requirements

### 9.1 Third-Party Integrations
- **Google Forms**: Membership form integration
- **Email Service**: Automated email notifications
- **SMS Service**: Mobile verification and alerts
- **Push Notifications**: FCM for Android, APNS for iOS
- **Analytics**: User behavior tracking

### 9.2 API Requirements
- **RESTful API**: Standard HTTP methods
- **API Documentation**: Swagger/OpenAPI specification
- **Rate Limiting**: Prevent abuse and ensure fair usage
- **Versioning**: Support multiple API versions
- **Error Handling**: Consistent error response format

## 10. Data Migration Strategy

### 10.1 Excel Data Migration (Future Phase)
- **Assessment**: Analyze existing Excel structure
- **Mapping**: Map Excel columns to database schema
- **Validation**: Data integrity checks
- **Import Tools**: Bulk import utilities
- **Testing**: Pilot migration with sample data

### 10.2 Migration Process
1. Export Excel data to CSV format
2. Data validation and cleanup
3. Database schema preparation
4. Staged data import (users, then transactions)
5. Data verification and reconciliation
6. User acceptance testing
7. Go-live coordination

## 11. Testing Requirements

### 11.1 Testing Types
- **Unit Testing**: Individual component testing
- **Integration Testing**: API and database integration
- **Mobile Testing**: iOS and Android device testing
- **Security Testing**: Vulnerability assessment
- **Performance Testing**: Load and stress testing
- **User Acceptance Testing**: End-user validation

### 11.2 Test Coverage
- **Minimum 80% code coverage**
- **All critical user journeys**
- **Payment processing workflows**
- **Admin functionality validation**
- **Cross-platform compatibility**

## 12. Deployment & Maintenance

### 12.1 Deployment Strategy
- **Staging Environment**: Pre-production testing
- **Blue-Green Deployment**: Zero-downtime deployments
- **Database Migrations**: Automated schema updates
- **Rollback Procedures**: Quick recovery from issues

### 12.2 Monitoring & Maintenance
- **Application Monitoring**: Real-time performance tracking
- **Error Logging**: Centralized error management
- **Database Monitoring**: Query performance optimization
- **Security Updates**: Regular security patch management
- **Backup Verification**: Regular backup restoration testing

## 13. Support & Documentation

### 13.1 User Documentation
- **Mobile App User Guide**
- **Admin Panel Documentation**
- **FAQ and Troubleshooting**
- **Video Tutorials**

### 13.2 Technical Documentation
- **API Documentation**
- **Database Schema Documentation**
- **Deployment Procedures**
- **System Architecture Documentation**

## 14. Success Metrics

### 14.1 Key Performance Indicators
- **User Adoption**: 90% of club members actively using the app
- **Payment Accuracy**: 99% accuracy in payment processing
- **System Uptime**: 99.9% availability
- **User Satisfaction**: >4.5 rating on app stores
- **Admin Efficiency**: 50% reduction in manual administrative tasks

### 14.2 Business Impact
- **Cost Reduction**: Eliminate Excel-based manual processes
- **Revenue Optimization**: Improved payment collection rates
- **Member Engagement**: Increased participation in club activities
- **Scalability**: Support for club expansion and new locations
