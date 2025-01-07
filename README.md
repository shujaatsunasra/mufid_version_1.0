# mufid_version_1.0

// Master Supabase Schema

// Table: users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT DEFAULT 'user', -- Roles: 'public', 'user', 'admin'
    verified BOOLEAN DEFAULT FALSE,
    profile_picture_url TEXT,
    phone_number TEXT UNIQUE, -- Added for OTP-based authentication
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);

-- Table: family_members
CREATE TABLE family_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id) ON DELETE CASCADE,
    first_name TEXT NOT NULL,
    last_name TEXT,
    dob DATE,
    relationship TEXT NOT NULL, -- parent, child, sibling, etc.
    parent_id UUID REFERENCES family_members (id), -- for hierarchical linking
    profile_picture_url TEXT NOT NULL,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    metadata JSONB -- For storing custom attributes dynamically
);

-- Table: community_tree
CREATE TABLE community_tree (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_member_id UUID REFERENCES family_members (id) ON DELETE CASCADE,
    approved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);

-- Table: stories
CREATE TABLE stories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    media_url TEXT NOT NULL,
    category TEXT NOT NULL,
    tags TEXT[],
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);

-- Table: otp_requests
CREATE TABLE otp_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number TEXT NOT NULL,
    otp_code TEXT NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now()
);

-- Table: role_requests
CREATE TABLE role_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id) ON DELETE CASCADE,
    requested_role TEXT NOT NULL, -- 'admin', 'editor', etc.
    status TEXT DEFAULT 'pending', -- 'pending', 'approved', 'rejected'
    reason TEXT,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);

-- Table: audit_logs
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users (id),
    action TEXT NOT NULL, -- Action performed
    target_id UUID, -- Affected resource
    details JSONB,
    created_at TIMESTAMP DEFAULT now()
);

-- Table: notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipient_id UUID REFERENCES users (id),
    message TEXT NOT NULL,
    seen BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT now()
);

// Functions and Triggers
-- Function to verify OTP and update status
CREATE FUNCTION verify_otp() RETURNS TRIGGER AS $$
BEGIN
    UPDATE otp_requests SET verified = TRUE WHERE id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger for OTP verification
CREATE TRIGGER verify_otp_trigger
AFTER UPDATE OF verified ON otp_requests
FOR EACH ROW
WHEN (NEW.verified = TRUE)
EXECUTE FUNCTION verify_otp();

-- Function to auto-update timestamps
CREATE FUNCTION update_timestamp() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers to auto-update timestamps
CREATE TRIGGER update_users_timestamp
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER update_family_members_timestamp
BEFORE UPDATE ON family_members
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER update_community_tree_timestamp
BEFORE UPDATE ON community_tree
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER update_stories_timestamp
BEFORE UPDATE ON stories
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER update_role_requests_timestamp
BEFORE UPDATE ON role_requests
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

// Indexes for performance optimization
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_phone_number ON users (phone_number);
CREATE INDEX idx_family_members_user_id ON family_members (user_id);
CREATE INDEX idx_family_members_parent_id ON family_members (parent_id);
CREATE INDEX idx_community_tree_family_member_id ON community_tree (family_member_id);
CREATE INDEX idx_stories_user_id ON stories (user_id);
CREATE INDEX idx_stories_tags ON stories USING gin (tags);
CREATE INDEX idx_audit_logs_user_id ON audit_logs (user_id);
CREATE INDEX idx_notifications_recipient_id ON notifications (recipient_id);

-- Ensure high scalability and dynamic adaptability for future features.
