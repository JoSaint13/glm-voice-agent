# Technical Specification

## 1. Technology Stack

### Frontend
- **Framework**: Next.js 14 with TypeScript
- **Build Tool**: Vite 5.x
- **State Management**: Zustand
- **Routing**: Next.js App Router
- **UI Library**: Tailwind CSS + Headless UI
- **Form Handling**: React Hook Form + Zod
- **Data Fetching**: TanStack Query
- **Key Dependencies**:
  * @tanstack/react-query - Server state management
  * react-hook-form - Form handling
  * zod - Schema validation
  * zustand - Client state management
  * @headlessui/react - Headless UI components
  * react-hot-toast - Toast notifications

### Backend
- **Runtime**: Node.js 20 LTS
- **Framework**: Next.js API Routes
- **Language**: TypeScript
- **Database**: PostgreSQL 15 with Prisma ORM
- **Authentication**: NextAuth.js with JWT
- **API Type**: REST with GraphQL for complex queries

### Infrastructure & DevOps
- **Hosting**: Vercel for frontend/edge functions, Railway for backend
- **Database Hosting**: Supabase
- **File Storage**: AWS S3 with Cloudfront CDN
- **CI/CD**: GitHub Actions
- **Monitoring**: Sentry for errors, Vercel Analytics for performance
- **Analytics**: PostHog (privacy-first)
- **Voice Services**: OpenAI Whisper + ElevenLabs API

## 2. System Architecture

### Architecture Pattern
Server-side rendered Next.js application with API routes for secure healthcare data handling.

### Component Diagram
```
┌───────────────────────────────────────────────────────────┐
│                  User's Device                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Next.js App (SSR)                      │   │
│  │  - Frontend Components                         │   │
│  │  - Authentication (NextAuth)                   │   │
│  │  - Voice Interaction (WebRTC)                  │   │
│  └─────────────────────┬─────────────────────────────┘   │
└───────────────────────┼───────────────────────────────────┘
                        │ HTTPS/WSS
                        ▼
┌───────────────────────────────────────────────────────────┐
│                 Edge Network (Vercel)                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Edge Functions                          │   │
│  │  - Voice Processing                             │   │
│  │  - API Gateway                                 │   │
│  └─────────────────────┬─────────────────────────────┘   │
└───────────────────────┼───────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│                 Backend Services                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │        API Routes (Node.js)                    │   │
│  │  - Business Logic                              │   │
│  │  - EHR Integration                             │   │
│  │  - Voice Processing                            │   │
│  └─────────────────────┬─────────────────────────────┘   │
└───────────────────────┼───────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│                 Data Layer                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────┐│
│  │   PostgreSQL     │  │   Redis Cache   │  │   S3     ││
│  │  (Clinical DB)  │  │  (Sessions)     │  │ (Files)  ││
│  └──────────────────┘  └──────────────────┘  └──────────┘│
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │         External Services                       │   │
│  │  - EHR APIs (Epic, Cerner)                      │   │
│  │  - OpenAI/Whisper (ASR)                        │   │
│  │  - ElevenLabs (TTS)                            │   │
│  └─────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

### Key Design Decisions
1. **HIPAA-Compliant Architecture**: All PHI data processed in-region, encrypted at rest and in transit
   - **Rationale**: Healthcare data has strict compliance requirements
   - **Trade-offs**: Higher infrastructure costs, limited third-party integrations

2. **Server-Side Rendering with Next.js**
   - **Rationale**: Better performance, improved SEO for healthcare content, secure session handling
   - **Trade-offs**: More server resources required, longer build times

3. **Modular Voice Service Architecture**
   - **Rationale**: Allows for switching between ASR/TTS providers as technology evolves
   - **Trade-offs**: Initial complexity in service abstraction

## 3. Data Models & Database Schema

### User Model
```typescript
interface User {
  id: string;                    // UUID
  email: string;                 // unique, indexed
  passwordHash: string;          // bcrypt
  name: string;
  avatar?: string;               // URL to avatar image
  role: 'admin' | 'clinician' | 'staff';
  organizationId: string;       // FK to Organization
  preferences: {
    language: string;            // ISO 639-1 code
    voiceModel?: string;         // Preferred TTS model
    timezone: string;
  };
  createdAt: Date;
  updatedAt: Date;
  lastLoginAt?: Date;
  isActive: boolean;
  twoFactorEnabled: boolean;
}
```

### Organization Model
```typescript
interface Organization {
  id: string;
  name: string;
  type: 'hospital' | 'clinic' | 'network';
  address: {
    street: string;
    city: string;
    state: string;
    zip: string;
    country: string;
  };
  ehrSystem?: 'epic' | 'cerner' | 'other';
  settings: {
    autoTranscription: boolean;
    voiceAuthentication: boolean;
    defaultLanguage: string;
    dialectSupport: string[];
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Patient Interaction Model
```typescript
interface PatientInteraction {
  id: string;
  patientId: string;            // FK to Patient
  clinicianId: string;          // FK to User
  sessionId: string;            // Unique voice session ID
  type: 'intake' | 'followup' | 'emergency';
  status: 'started' | 'in_progress' | 'completed' | 'failed';
  duration?: number;            // in seconds
  recordingUrl?: string;        // Encrypted URL to recording
  transcript?: string;          // Processed transcript
  summary?: string;             // AI-generated summary
  extractedData?: {              // Structured data extracted from voice
    symptoms: string[];
    medications: string[];
    allergies: string[];
    vitalSigns?: {
      bloodPressure?: string;
      heartRate?: number;
      temperature?: number;
    };
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Patient Model
```typescript
interface Patient {
  id: string;
  mrn: string;                  // Medical Record Number
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  gender: 'male' | 'female' | 'other';
  contactInfo: {
    email?: string;
    phone?: string;
    address?: {
      street: string;
      city: string;
      state: string;
      zip: string;
    };
  };
  language: string;             // Primary language
  preferredLanguage?: string;   // Preferred for interactions
  emergencyContact?: {
    name: string;
    relationship: string;
    phone: string;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### EHR Integration Model
```typescript
interface EHRIntegration {
  id: string;
  organizationId: string;       // FK to Organization
  ehrSystem: 'epic' | 'cerner' | 'other';
  apiEndpoint: string;
  credentials: {
    apiKey?: string;
    secret?: string;
    oauthToken?: string;
  };
  lastSyncAt?: Date;
  settings: {
    syncFrequency: 'realtime' | 'hourly' | 'daily';
    autoCreatePatients: boolean;
    syncDemographics: boolean;
    syncClinicalData: boolean;
  };
}
```

### Database Indexes
```sql
-- Critical indexes for performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_org_id ON users(organizationId);
CREATE INDEX idx_patient_interactions_session ON patientInteractions(sessionId);
CREATE INDEX idx_patient_interactions_patient ON patientInteractions(patientId);
CREATE INDEX idx_patient_interactions_clinician ON patientInteractions(clinicianId);
CREATE INDEX idx_patients_mrn ON patients(mrn);
CREATE INDEX idx_ehr_integration_org ON ehrIntegrations(organizationId);
```

## 4. API Design

### Authentication Endpoints
```
POST   /api/auth/signup                - Create new user account
POST   /api/auth/login                 - Login with email/password
POST   /api/auth/logout                - Invalidate session
GET    /api/auth/me                    - Get current user
POST   /api/auth/reset                 - Password reset request
POST   /api/auth/verify-reset          - Verify reset token
PUT    /api/auth/password              - Change password
POST   /api/auth/2fa/setup            - Setup 2FA
POST   /api/auth/2fa/verify           - Verify 2FA code
```

### Organization Management Endpoints
```
GET    /api/organizations              - List organizations
POST   /api/organizations              - Create new organization
GET    /api/organizations/:id         - Get organization
PUT    /api/organizations/:id         - Update organization
DELETE /api/organizations/:id         - Delete organization
POST   /api/organizations/:id/ehr     - Configure EHR integration
```

### Patient Management Endpoints
```
GET    /api/patients                  - List patients (paginated)
POST   /api/patients                  - Create new patient
GET    /api/patients/:id              - Get patient
PUT    /api/patients/:id              - Update patient
DELETE /api/patients/:id              - Delete patient
POST   /api/patients/search           - Search patients
POST   /api/patients/:id/interactions - Get patient interactions
```

### Voice Interaction Endpoints
```
POST   /api/voice/sessions            - Create new voice session
WebSocket /api/voice/:session/stream  - Voice streaming
GET    /api/voice/sessions/:id        - Get session details
POST   /api/voice/sessions/:id/transcript - Request transcript
GET    /api/voice/sessions/:id/transcript - Get transcript
POST   /api/voice/sessions/:id/summary - Request summary
GET    /api/voice/sessions/:id/summary - Get summary
```

### EHR Integration Endpoints
```
GET    /api/ehr/sync/:orgId           - Trigger EHR sync
GET    /api/ehr/status/:orgId         - Get EHR sync status
POST   /api/ehr/patients/:id          - Push patient data to EHR
GET    /api/ehr/patients/:mrn         - Get patient from EHR
```

### Request/Response Examples

**Create Voice Session:**
```http
POST /api/voice/sessions
Content-Type: application/json
Authorization: Bearer <token>

{
  "patientId": "uuid-here",
  "type": "intake",
  "language": "en-US",
  "voiceModel": "eleven-labs-premium"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "voice-session-uuid",
    "status": "started",
    "wsUrl": "wss://api.medivoice.com/voice/voice-session-uuid/stream",
    "expiresAt": "2025-01-15T11:00:00Z"
  }
}
```

**Patient Interaction with Transcription:**
```http
GET /api/patients/uuid-here/interactions
Authorization: Bearer <token>

Response:
{
  "success": true,
  "data": {
    "interactions": [
      {
        "id": "interaction-uuid",
        "sessionId": "voice-session-uuid",
        "type": "intake",
        "status": "completed",
        "duration": 345,
        "transcript": "Hello, my name is John and I'm here for my annual checkup...",
        "summary": "Annual checkup for John Doe, no current complaints, medications stable.",
        "extractedData": {
          "symptoms": [],
          "medications": ["Lisinopril 10mg daily", "Atorvastatin 20mg daily"],
          "vitalSigns": {
            "bloodPressure": "120/80",
            "heartRate": 72,
            "temperature": 98.6
          }
        },
        "createdAt": "2025-01-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 5
    }
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "RECORDING_NOT_FOUND",
    "message": "The requested recording could not be found",
    "details": {
      "sessionId": "voice-session-uuid",
      "attemptedAt": "2025-01-15T11:05:00Z"
    },
    "requestId": "req-uuid",
    "timestamp": "2025-01-15T11:05:15Z"
  }
}
```

## 5. Frontend Architecture

### Project Structure
```
/
├── src/
│   ├── app/                         # Next.js app directory
│   │   ├── (auth)/                 # Auth-related pages
│   │   │   ├── login/
│   │   │   ├── signup/
│   │   │   └── forgot-password/
│   │   ├── (dashboard)/            # Protected dashboard pages
│   │   │   ├── dashboard/
│   │   │   ├── patients/
│   │   │   ├── interactions/
│   │   │   ├── settings/
│   │   │   └── organizations/
│   │   └── api/                    # API routes
│   ├── components/
│   │   ├── ui/                     # Reusable UI components
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── dialog.tsx
│   │   │   └── ...
│   │   ├── features/               # Feature-specific components
│   │   │   ├── voice/
│   │   │   │   ├── VoiceSession.tsx
│   │   │   │   ├── TranscriptViewer.tsx
│   │   │   │   └── WaveformVisualizer.tsx
│   │   │   ├── patients/
│   │   │   │   ├── PatientList.tsx
│   │   │   │   ├── PatientForm.tsx
│   │   │   │   └── PatientDetails.tsx
│   │   │   └── auth/
│   │   │       ├── LoginForm.tsx
│   │   │       └── SignupForm.tsx
│   │   └── layouts/                # Layout components
│   │       ├── AuthLayout.tsx
│   │       ├── DashboardLayout.tsx
│   │       └── SettingsLayout.tsx
│   ├── lib/
│   │   ├── api.ts                  # API client setup
│   │   ├── auth.ts                 # Auth utilities
│   │   ├── validation.ts           # Zod schemas
│   │   ├── utils.ts                # Helper functions
│   │   ├── voice.ts                # Voice processing utilities
│   │   └── ehr.ts                  # EHR integration helpers
│   ├── hooks/                      # Custom React hooks
│   │   ├── useAuth.ts
│   │   ├── usePatients.ts
│   │   └── useVoiceSession.ts
│   ├── store/                      # Zustand stores
│   │   ├── auth.ts
│   │   ├── app.ts
│   │   └── voice.ts
│   ├── types/                      # TypeScript types/interfaces
│   │   ├── auth.ts
│   │   ├── patient.ts
│   │   ├── voice.ts
│   │   └── ehr.ts
│   └── styles/                     # Global styles
│       ├── globals.css
│       └── tailwind.css
├── public/                         # Static assets
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── docs/                           # Additional documentation
```

### State Management Strategy
- **Server State**: Use TanStack Query for all API data with prefetching
- **UI State**: Use Zustand for complex UI state (voice session state, UI preferences)
- **Form State**: React Hook Form with Zod validation schemas
- **Auth State**: NextAuth.js session with custom auth context for React access

### Component Patterns
```typescript
// Example: VoiceSession component with proper patterns
import { useVoiceSession } from '@/hooks/useVoiceSession';
import { useQueryClient } from '@tanstack/react-query';

export function VoiceSession({ sessionId }) {
  const queryClient = useQueryClient();
  const { 
    status, 
    transcript, 
    isRecording, 
    startRecording, 
    stopRecording,
    error 
  } = useVoiceSession(sessionId);

  const handleComplete = async () => {
    await stopRecording();
    // Invalidate and refetch interaction data
    queryClient.invalidateQueries(['patient-interactions']);
  };

  if (error) {
    return <ErrorMessage error={error} />;
  }

  return (
    <div className="voice-session">
      <div className="waveform-container">
        {/* Waveform visualization */}
      </div>
      
      <div className="transcript-container">
        <TranscriptViewer transcript={transcript} />
      </div>
      
      <div className="controls">
        <Button 
          onClick={isRecording ? stopRecording : startRecording}
          variant={isRecording ? 'destructive' : 'default'}
        >
          {isRecording ? 'Stop Recording' : 'Start Recording'}
        </Button>
        
        {status === 'recording' && (
          <Button onClick={handleComplete}>
            Complete Session
          </Button>
        )}
      </div>
    </div>
  );
}
```

## 6. Security Implementation

### Authentication Flow
1. User submits credentials through secure HTTPS connection
2. Backend validates using NextAuth.js with custom credentials provider
3. Upon success, JWT is generated and stored in httpOnly, secure cookie
4. Frontend uses the cookie for subsequent authenticated requests
5. Backend validates JWT on all protected routes
6. Optional: Two-factor authentication verification

### Security Checklist
- [ ] Input validation on all forms (Zod schemas)
- [ ] SQL injection prevention (Prisma ORM with parameterized queries)
- [ ] XSS protection (React escapes by default, sanitize HTML if needed)
- [ ] CSRF tokens for state-changing operations (NextAuth built-in)
- [ ] Rate limiting on auth endpoints (5 attempts/15 min)
- [ ] Password strength requirements (min 12 chars, 1 uppercase, 1 number, 1 special char)
- [ ] HTTPS only in production (enforced by Next.js)
- [ ] Secure headers (CSP, X-Frame-Options, etc.) via Next.js config
- [ ] Regular dependency updates (GitHub Dependabot)
- [ ] Audit logging for sensitive operations
- [ ] HIPAA-compliant data encryption at rest (AES-256)
- [ ] PHI data encrypted in transit (TLS 1.3)
- [ ] Access control based on user roles and organization
- [ ] Session timeout (30 minutes of inactivity)
- [ ] Secure API key storage (environment variables with encryption)

### Voice Security
- Real-time encryption of voice streams using WebRTC with SRTP
- Secure storage of recordings with encrypted filenames and restricted access
- Voice biometric authentication option (when enabled)
- Automatic redaction of PHI from transcripts if requested

## 7. Performance Optimization

### Frontend Performance
- **Code Splitting**: Route-based splitting with Next.js dynamic imports
- **Image Optimization**: Next.js Image component with WebP and AVIF support
- **Bundle Size**: Keep main bundle < 150KB for optimal loading
- **Caching**: Aggressive caching of static assets and API responses
- **Core Web Vitals Targets**:
  * LCP < 2.0s (healthcare content load time critical)
  * FID < 80ms (important for clinical workflow)
  * CLS < 0.05 (minimal layout shift for professional interface)

### Backend Performance
- **Database Query Optimization**: Use indexes, avoid N+1 queries with Prisma
- **Caching Strategy**: Redis for session data and frequently accessed patient data
- **API Response Times**: Target < 200ms for p95 (critical for clinical workflow)
- **Voice Processing**: Edge functions for initial voice processing, reduce latency

### Voice Processing Optimization
- WebRTC for direct client-server voice streaming
- Chunk-based processing for large audio files
- Background transcription to minimize user wait time
- CDN distribution of TTS audio files

### Monitoring
- Real User Monitoring (RUM) with Vercel Analytics
- Error tracking with Sentry
- Performance budgets in CI/CD
- Custom metrics for voice session quality and latency

## 8. Testing Strategy

### Unit Tests
- **Tool**: Vitest + Testing Library
- **Coverage Target**: > 80% for critical business logic
- **Focus**: Utils, hooks, pure functions, voice processing logic

### Integration Tests
- **Tool**: Vitest + MSW (Mock Service Worker)
- **Focus**: API integration, data fetching logic, authentication flows
- **Voice Integration**: Mock voice API responses

### End-to-End Tests
- **Tool**: Playwright
- **Critical Paths to Test**:
  * User signup/login flow
  * Patient creation and search
  * Voice session creation and interaction
  * EHR data synchronization
  * HIPAA compliance requirements

### Test Example
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { VoiceSession } from '@/components/voice/VoiceSession';

// Mock the voice hook
vi.mock('@/hooks/useVoiceSession', () => ({
  useVoiceSession: () => ({
    status: 'ready',
    transcript: '',
    isRecording: false,
    startRecording: vi.fn(),
    stopRecording: vi.fn(),
  }),
}));

const createQueryClient = () => new QueryClient({
  defaultOptions: {
    queries: { retry: false },
  },
});

describe('VoiceSession', () => {
  it('renders controls correctly', () => {
    render(
      <QueryClientProvider client={createQueryClient()}>
        <VoiceSession sessionId="test-session" />
      </QueryClientProvider>
    );
    
    expect(screen.getByText('Start Recording')).toBeInTheDocument();
  });

  it('starts recording when button clicked', async () => {
    const { result } = render(
      <QueryClientProvider client={createQueryClient()}>
        <VoiceSession sessionId="test-session" />
      </QueryClientProvider>
    );
    
    fireEvent.click(screen.getByText('Start Recording'));
    
    expect(result.current.startRecording).toHaveBeenCalled();
  });
});
```

### Compliance Testing
- Regular penetration testing for HIPAA compliance
- Automated scanning for PHI exposure
- Privacy impact assessments for new features
- Accessibility testing (WCAG 2.1 AA) for healthcare users

## 9. Deployment & DevOps

### Environment Setup
```env
# .env.local (development)
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=dev-secret-change-in-prod
DATABASE_URL=postgresql://localhost:5432/medivoice_dev
REDIS_URL=redis://localhost:6379
AWS_S3_BUCKET=medivoice-dev
AWS_ACCESS_KEY_ID=dev-key
AWS_SECRET_ACCESS_KEY=dev-secret
OPENAI_API_KEY=dev-key
ELEVENLABS_API_KEY=dev-key

# .env.production
NEXTAUTH_URL=https://app.medivoice.com
NEXTAUTH_SECRET=secure-production-secret
DATABASE_URL=postgresql://user:pass@prod-db:5432/medivoice_prod
REDIS_URL=redis://prod-redis:6379
AWS_S3_BUCKET=medivoice-prod
AWS_ACCESS_KEY_ID=prod-key
AWS_SECRET_ACCESS_KEY=prod-secret
OPENAI_API_KEY=prod-key
ELEVENLABS_API_KEY=prod-key
```

### Deployment Pipeline
1. **Development**: Local dev server with hot reload
2. **Staging**: Auto-deploy from `develop` branch to Vercel staging
3. **Production**: Deploy from `main` branch after PR approval and automated tests

### GitHub Actions Workflow
```yaml
name: CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run build

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      - uses: vercel/action@v1
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          
  deploy-production:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      - uses: vercel/action@v1
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel