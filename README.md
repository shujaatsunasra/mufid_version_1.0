Master Supabase Schema and Workflow
1. users Table
sql
Copy code
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT DEFAULT 'public', -- public, user, admin, super_admin
    verified BOOLEAN DEFAULT FALSE,
    phone_number TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    metadata JSONB, -- Reserved for user-specific preferences
    is_blacklisted BOOLEAN DEFAULT FALSE -- For banning misuse
);
2. family_members Table
Includes a flag system for validating relationships dynamically with multi-level checks.

sql
Copy code
CREATE TABLE family_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id) ON DELETE CASCADE,
    first_name TEXT NOT NULL,
    last_name TEXT,
    dob DATE NOT NULL,
    relationship TEXT NOT NULL, -- parent, sibling, child, etc.
    parent_id UUID REFERENCES family_members (id), -- For hierarchical relationships
    profile_picture_url TEXT NOT NULL,
    otp_verified BOOLEAN DEFAULT FALSE,
    is_flagged BOOLEAN DEFAULT FALSE, -- To flag suspicious entries
    consensus_score INT DEFAULT 0, -- Aggregated score for validation
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    metadata JSONB -- Custom family attributes
);
3. flagged_entries Table (Misuse Detection Layer)
Tracks flagged users and relationships, enabling consensus voting for decision-making.

sql
Copy code
CREATE TABLE flagged_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flagged_by UUID REFERENCES users (id), -- Who flagged it
    family_member_id UUID REFERENCES family_members (id),
    reason TEXT NOT NULL, -- Description of the misuse
    status TEXT DEFAULT 'pending', -- pending, resolved, rejected
    consensus_votes INT DEFAULT 0, -- Aggregated votes for action
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);
4. otp_requests Table
Enhanced OTP with rate-limiting and reputation scoring to track potential misuse.

sql
Copy code
CREATE TABLE otp_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number TEXT NOT NULL,
    otp_code TEXT NOT NULL,
    request_count INT DEFAULT 0, -- Prevent abuse by tracking requests
    reputation_score INT DEFAULT 100, -- Decreases on misuse
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now(),
    expires_at TIMESTAMP NOT NULL -- OTP expiry timestamp
);
5. audit_logs Table
Tracks every action with high granularity to detect patterns of misuse.

sql
Copy code
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id),
    action TEXT NOT NULL, -- Action: added_member, flagged_member, etc.
    target_id UUID, -- Affected resource
    details JSONB, -- JSON payload with full details
    created_at TIMESTAMP DEFAULT now()
);
6. community_tree Table
Public and private data segregation with granular access policies.

sql
Copy code
CREATE TABLE community_tree (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_member_id UUID REFERENCES family_members (id) ON DELETE CASCADE,
    approved BOOLEAN DEFAULT FALSE, -- Requires admin approval
    is_public BOOLEAN DEFAULT TRUE, -- Public visibility
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);
Workflow for Misuse Mitigation
1. Dynamic Validation Layer
Use AI-based rules stored in Supabase Edge Functions:
Cross-check relationships for logical inconsistencies (e.g., age gaps, duplicate entries).
Validate relationship hierarchy dynamically (e.g., a child cannot be older than a parent).
Reject implausible entries instantly.
2. Flagging and Consensus Mechanism
Auto-flag suspicious data:
Inconsistent dob or impossible relationships.
High frequency of edits or entries by a single user.
Rate-limit OTP requests to reduce spamming.
Use the flagged_entries table to allow trusted family members and admins to vote on flagged data:
A threshold of consensus_votes determines whether an entry is rejected or retained.
Provide transparency logs to voters to avoid bias.
3. Hierarchical Role Approvals
Users with the admin role oversee flagged entries.
Escalate unresolved flags to super_admin for final decisions.
Automatically blacklist malicious users after repeated violations (tracked in is_blacklisted).
4. Real-Time Notifications
Notify family members when:
A new relationship is added.
Someone requests to join their family.
Their relationship is flagged for review.
Notifications logged in a notifications table:
sql
Copy code
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipient_id UUID REFERENCES users (id),
    message TEXT NOT NULL,
    seen BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now()
);
5. Privacy and Audit Layers
Community View: Public family trees are readable without authentication but omit sensitive data (e.g., dob or contact info).
Full family editing is OTP-protected, verified through cross-validation with family_members.
6. Intelligent Recovery
Allow users to dispute blacklisting or rejected entries through an appeals system stored in flagged_entries.
Security Enhancements
Encrypt sensitive fields like phone_number and otp_code with AES256.
Use Supabase RLS (Row-Level Security) to enforce:
Public view-only access for community_tree.
Scoped access for users to edit their family.
Implement request throttling for OTP and family edits.
Benefits of the Design
Misuse Resilience: Layers of checks, flagging, and consensus prevent false entries from spreading.
Auditability: Every action is logged, providing a transparent misuse trail.
Dynamic Validation: AI ensures family relationships follow logical rules.
Privacy Respect: Public data is segmented, ensuring sensitive fields stay private.
Scalability: Decoupled tables, triggers, and functions allow for seamless growth.
