# Data Entry/Migration Plan for RHS Soccer System

Based on the analysis of the data models and their dependencies, this document outlines a comprehensive data entry/migration plan organized by dependency order and minimum viable data requirements.

## Phase 1: Foundation Setup (System Configuration)

### 1.1 Payment Accounts

**Priority:** Critical (needed for team fees)
**Minimum Data:**

- Create at least one PaymentAccount (e.g., "Booster Club")
  - `name`: "RHS Soccer Booster Club"
  - `account_type`: "booster_club"
  - `stripe_publishable_key`: Test key initially
  - `stripe_secret_key`: Test key initially
  - `is_active`: true
  - `is_default`: true
  - `organization_name`: "Rosemount High School Booster Club"
  - `contact_email`: Admin email

### 1.2 Roles & Permissions

**Priority:** Critical (RBAC foundation)
**Setup Method:** Automated via management command

Run the following command to set up the complete RBAC system:

```bash
docker compose run --rm django python manage.py setup_rbac
```

This command will automatically create:

- **9 Roles** with proper hierarchy:

  - `super_admin` (level: 1) - Full system access
  - `club_admin` (level: 2) - Club-wide administration
  - `head_coach` (level: 3, team_specific: true) - Full team management
  - `assistant_coach` (level: 4, team_specific: true) - Limited team management
  - `team_manager` (level: 5, team_specific: true) - Team administrative support
  - `player` (level: 6, team_specific: true) - Player access
  - `parent` (level: 7) - Parent/guardian access
  - `treasurer` (level: 2) - Financial management
  - `volunteer_coordinator` (level: 3) - Volunteer management

- **78 Permissions** across all categories:

  - User & Role Management (11 permissions)
  - Team Management (8 permissions)
  - Player Management (8 permissions)
  - Match Management (12 permissions)
  - Event Management (12 permissions)
  - Financial Management (11 permissions)
  - Tryout Management (6 permissions)
  - Communication (7 permissions)
  - General (3 permissions)

- **251 Role-Permission Assignments** automatically configured based on role responsibilities

**Note:** The command is idempotent - running it multiple times will update existing data rather than creating duplicates. Use `--reset` flag to clear and rebuild from scratch if needed.

## Phase 2: User Accounts

### 2.1 Administrator Account

**Priority:** Critical
**Minimum Data:**

- Create superuser account
- Assign `admin` role via UserRole

### 2.2 Coach Accounts

**Priority:** High
**Minimum Data per coach:**

- User record: `email`, `first_name`, `last_name`
- CoachProfile: `license_level`, `years_experience`
- UserRole: Assign `head_coach` or `assistant_coach` role

## Phase 3: Teams & Season Setup

### 3.1 Club Teams

**Priority:** Critical
**Minimum Data per team:**

- `name`: "Varsity Boys"
- `level`: "varsity"
- `season`: "Fall 2024" or current season
- `is_active`: true
- `payment_account`: Link to created PaymentAccount
- `max_players`: 25

### 3.2 Team Fees (Optional initially)

**Minimum Data per fee:**

- `team`: Link to ClubTeam
- `amount`: Fee amount
- `description`: "Season registration fee"
- `due_date`: Season start date

## Phase 4: Player & Family Registration

### 4.1 Family Registration Flow (Recommended)

**For each family:**

1. **Create FamilyRegistration record:**

   - `team`: Target team
   - `primary_parent_email`: Parent email
   - `invited_by`: Coach user
   - `player_first_name`, `player_last_name`
   - `status`: "pending"

2. **When parent completes registration:**
   - Create parent User account
   - Create ParentProfile
   - Create player User account
   - Create PlayerProfile with:
     - `jersey_number` (if assigned)
     - `position`
     - `graduation_year`
     - `school_grade`
   - Create ParentChildRelationship
   - Create TeamMembership linking player to team
   - Assign roles via UserRole

## Phase 5: Competition Setup

### 5.1 Venues

**Minimum Data:**

- Home field venue:
  - `name`: "RHS Soccer Field"
  - `venue_type`: "field"
  - `address`: School address
  - `is_home_venue`: true

### 5.2 Opposing Teams

**Minimum Data per team:**

- `name`: Team name
- `school_or_club`: School name
- `city`, `state`

### 5.3 Matches

**Minimum Data per match:**

- `home_team`: ClubTeam reference
- `away_team`: OpposingTeam reference
- `match_type`: "friendly" or "league"
- `date_time`: Match datetime
- `venue`: Venue reference
- `status`: "scheduled"

## Phase 6: Optional/Enhancement Data

### 6.1 Events & Activities

- Team practices via TeamActivity
- Club events via Event model
- Team meetings via TeamEvent

### 6.2 Fundraising

- Create FundraisingCategory records
- Set up Campaign if needed

### 6.3 Historical Data

- SeasonHistory for returning players
- Past match results

## Migration Sequence Summary

### Week 1: System Setup

1. Payment accounts (manual setup via Django Admin)
2. Roles & permissions (automated via `setup_rbac` command)
3. Admin user creation (via `createsuperuser` command)
4. Coach accounts (manual creation and role assignment)

### Week 2: Team Formation

1. Create teams for current season
2. Set up team fees
3. Configure venues

### Week 3-4: Player Registration

1. Send family invitations
2. Process registrations as they come in
3. Create player/parent accounts
4. Establish relationships

### Week 5: Competition Setup

1. Add opposing teams
2. Schedule matches
3. Set up events/activities

## Minimum Viable Dataset

For a functional system with one team:

- **1** PaymentAccount
- **9** Roles (created automatically via `setup_rbac` command)
- **78** Permissions (created automatically via `setup_rbac` command)
- **251** Role-Permission assignments (created automatically via `setup_rbac` command)
- **1** Admin user
- **1-2** Coach users with profiles
- **1** ClubTeam
- **10-15** Players with profiles
- **10-20** Parents with profiles and relationships
- **1** Home venue
- **5-10** Opposing teams
- **5-10** Scheduled matches

## Data Entry Methods

### Manual Entry via Django Admin

Best for:

- Initial system configuration (roles, permissions)
- Payment account setup
- Creating admin and coach accounts

### Bulk Import via CSV

Best for:

- Returning families from previous seasons
- Opposing teams list
- Match schedules

### Self-Service Registration

Best for:

- New family registrations
- Parent account creation
- Player profile completion

## Validation Checkpoints

### After Phase 1

- [ ] Payment processing test transaction
- [ ] Role assignment verification
- [ ] Permission checks for each role

### After Phase 2

- [ ] Admin can access all areas
- [ ] Coaches can access team management

### After Phase 3

- [ ] Teams appear in system
- [ ] Fee structure is correct

### After Phase 4

- [ ] Parent-child relationships established
- [ ] Players assigned to correct teams
- [ ] Roster counts accurate

### After Phase 5

- [ ] Match schedule visible
- [ ] Venues properly configured
- [ ] Opposing team data complete

## Rollback Strategy

Each phase should be completed in a transaction-safe manner:

1. Backup database before each phase
2. Test in staging environment first
3. Maintain audit log of all data entries
4. Keep original data sources (CSVs, forms) for reference

## Success Metrics

- **Phase 1-2**: System accessible, users can log in
- **Phase 3**: Teams visible, fees calculable
- **Phase 4**: 80% registration completion rate
- **Phase 5**: Full season schedule published
- **Overall**: System operational for season start

## Notes

- Start with test Stripe keys, switch to production after validation
- Consider running parallel with old system for first 2 weeks
- Daily backups during migration period
- Assign dedicated support during registration period
- Document any custom data transformations needed

This plan ensures proper dependency ordering and provides the minimum data needed for a functional soccer management system while allowing for incremental addition of features and data.
