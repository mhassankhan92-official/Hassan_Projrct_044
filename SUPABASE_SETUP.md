# Supabase Integration Setup Guide

## ✅ Completed Implementation

### 1. Core Setup

- ✅ Installed `@supabase/supabase-js` package
- ✅ Created environment configuration (`.env.local`)
- ✅ Created Supabase client configuration (`src/lib/supabase.ts`)
- ✅ Created complete TypeScript database types (`src/lib/database.types.ts`)

### 2. Authentication System

- ✅ Created `AuthContext` and `AuthProvider` (`src/contexts/AuthContext.tsx`)
- ✅ Created `ProtectedRoute` component with role-based access control
- ✅ Updated `Login.tsx` with Supabase authentication
- ✅ Updated `App.tsx` with AuthProvider and protected routes

### 3. Data Services

Created service files in `src/lib/services/`:

- ✅ `studentsService.ts` - Full CRUD operations
- ✅ `teachersService.ts` - Full CRUD operations
- ✅ `classesService.ts` - Full CRUD operations
- ✅ `attendanceService.ts` - Full CRUD + statistics
- ✅ `timetableService.ts` - Full CRUD operations
- ✅ `announcementsService.ts` - Full CRUD + real-time subscriptions
- ✅ `storageService.ts` - File upload/delete operations

### 4. React Query Hooks

Created custom hooks in `src/hooks/`:

- ✅ `useStudents.ts`
- ✅ `useTeachers.ts`
- ✅ `useClasses.ts`
- ✅ `useAttendance.ts`
- ✅ `useTimetable.ts`
- ✅ `useAnnouncements.ts` (with real-time subscriptions)

---

## 🚀 Next Steps: Supabase Dashboard Setup

### Step 1: Create Supabase Project

1. Go to [supabase.com](https://supabase.com)
2. Sign up or log in
3. Click "New Project"
4. Fill in:
   - **Project Name**: e.g., "EduManage"
   - **Database Password**: Create a strong password (save it!)
   - **Region**: Choose closest to you
5. Wait for project to be created (~2 minutes)

### Step 2: Get Your Credentials

1. Go to **Project Settings** (⚙️ icon)
2. Click **API** in the sidebar
3. Copy these values:
   - **Project URL** → `VITE_SUPABASE_URL`
   - **anon public** key → `VITE_SUPABASE_ANON_KEY`

### Step 3: Update Environment Variables

Edit `.env.local` file with your actual credentials:

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key-here
```

### Step 4: Create Database Tables

Go to **SQL Editor** in Supabase dashboard and run these SQL commands:

#### 1. Create Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL UNIQUE,
  role TEXT NOT NULL CHECK (role IN ('admin', 'teacher', 'student')),
  full_name TEXT NOT NULL,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policies
CREATE POLICY "Users can view all users" ON users
  FOR SELECT USING (true);

CREATE POLICY "Users can update their own profile" ON users
  FOR UPDATE USING (auth.uid() = id);
```

#### 2. Create Students Table

```sql
CREATE TABLE students (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  enrollment_number TEXT NOT NULL UNIQUE,
  class_id UUID REFERENCES classes(id),
  date_of_birth DATE NOT NULL,
  gender TEXT NOT NULL,
  address TEXT,
  parent_name TEXT,
  parent_contact TEXT,
  parent_email TEXT,
  emergency_contact TEXT,
  blood_group TEXT,
  medical_conditions TEXT,
  enrollment_date DATE DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE students ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Students visible to authenticated users" ON students
  FOR SELECT TO authenticated USING (true);

CREATE POLICY "Admins and teachers can manage students" ON students
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid()
      AND users.role IN ('admin', 'teacher')
    )
  );
```

#### 3. Create Teachers Table

```sql
CREATE TABLE teachers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  employee_id TEXT NOT NULL UNIQUE,
  department TEXT NOT NULL,
  subject_specialization TEXT[] NOT NULL,
  qualification TEXT NOT NULL,
  experience_years INTEGER NOT NULL DEFAULT 0,
  date_of_joining DATE DEFAULT CURRENT_DATE,
  phone TEXT,
  address TEXT,
  emergency_contact TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE teachers ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Teachers visible to authenticated users" ON teachers
  FOR SELECT TO authenticated USING (true);

CREATE POLICY "Only admins can manage teachers" ON teachers
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid() AND users.role = 'admin'
    )
  );
```

#### 4. Create Classes Table

```sql
CREATE TABLE classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  grade INTEGER NOT NULL,
  section TEXT NOT NULL,
  class_teacher_id UUID REFERENCES teachers(id),
  room_number TEXT,
  capacity INTEGER NOT NULL,
  academic_year TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(grade, section, academic_year)
);

ALTER TABLE classes ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Classes visible to authenticated users" ON classes
  FOR SELECT TO authenticated USING (true);

CREATE POLICY "Admins and teachers can manage classes" ON classes
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid()
      AND users.role IN ('admin', 'teacher')
    )
  );
```

#### 5. Create Attendance Table

```sql
CREATE TABLE attendance (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
  class_id UUID NOT NULL REFERENCES classes(id),
  date DATE NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('present', 'absent', 'late', 'excused')),
  remarks TEXT,
  marked_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(student_id, date)
);

ALTER TABLE attendance ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Attendance visible to authenticated users" ON attendance
  FOR SELECT TO authenticated USING (true);

CREATE POLICY "Teachers and admins can manage attendance" ON attendance
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid()
      AND users.role IN ('admin', 'teacher')
    )
  );
```

#### 6. Create Timetable Table

```sql
CREATE TABLE timetable (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  subject TEXT NOT NULL,
  teacher_id UUID NOT NULL REFERENCES teachers(id),
  day_of_week INTEGER NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  room_number TEXT,
  academic_year TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE timetable ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Timetable visible to authenticated users" ON timetable
  FOR SELECT TO authenticated USING (true);

CREATE POLICY "Admins and teachers can manage timetable" ON timetable
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid()
      AND users.role IN ('admin', 'teacher')
    )
  );
```

#### 7. Create Announcements Table

```sql
CREATE TABLE announcements (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  author_id UUID NOT NULL REFERENCES users(id),
  is_important BOOLEAN DEFAULT false,
  target_roles TEXT[] DEFAULT '{"admin","teacher","student"}',
  attachment_url TEXT,
  published_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE announcements ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Announcements visible to authenticated users" ON announcements
  FOR SELECT TO authenticated USING (
    published_at IS NOT NULL AND published_at <= NOW() AND
    (expires_at IS NULL OR expires_at > NOW())
  );

CREATE POLICY "Admins and teachers can manage announcements" ON announcements
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM users WHERE users.id = auth.uid()
      AND users.role IN ('admin', 'teacher')
    )
  );
```

### Step 5: Create Storage Buckets

Go to **Storage** in Supabase dashboard:

1. Create **avatars** bucket:

   - Click "New Bucket"
   - Name: `avatars`
   - Public bucket: ✅ Yes
   - Click "Create Bucket"

2. Create **announcements** bucket:
   - Click "New Bucket"
   - Name: `announcements`
   - Public bucket: ✅ Yes
   - Click "Create Bucket"

### Step 6: Set Up Storage Policies

For each bucket, add these policies:

**Avatars Bucket Policies:**

```sql
-- Allow authenticated users to upload
CREATE POLICY "Allow authenticated uploads" ON storage.objects
  FOR INSERT TO authenticated
  WITH CHECK (bucket_id = 'avatars');

-- Allow users to update their own avatar
CREATE POLICY "Allow users to update own avatar" ON storage.objects
  FOR UPDATE TO authenticated
  USING (bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]);

-- Allow public access
CREATE POLICY "Public access to avatars" ON storage.objects
  FOR SELECT TO public
  USING (bucket_id = 'avatars');

-- Allow users to delete their own avatar
CREATE POLICY "Allow users to delete own avatar" ON storage.objects
  FOR DELETE TO authenticated
  USING (bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]);
```

**Announcements Bucket Policies:**

```sql
-- Allow admins/teachers to upload
CREATE POLICY "Allow uploads for admins/teachers" ON storage.objects
  FOR INSERT TO authenticated
  WITH CHECK (
    bucket_id = 'announcements' AND
    EXISTS (SELECT 1 FROM users WHERE users.id = auth.uid() AND users.role IN ('admin', 'teacher'))
  );

-- Public read access
CREATE POLICY "Public access to announcements" ON storage.objects
  FOR SELECT TO public
  USING (bucket_id = 'announcements');
```

---

## 📝 Test Your Setup

### 1. Create a Test Admin User

Run in SQL Editor:

```sql
-- This creates an auth user and profile
-- Replace email and password with your test credentials
INSERT INTO auth.users (email, encrypted_password, email_confirmed_at)
VALUES ('admin@test.com', crypt('password123', gen_salt('bf')), NOW());

-- Get the user ID (will be displayed in results)
-- Then insert into users table
INSERT INTO users (id, email, role, full_name)
VALUES (
  'paste-user-id-here',
  'admin@test.com',
  'admin',
  'Test Admin'
);
```

### 2. Test Login

1. Run your app: `npm run dev`
2. Go to `/login`
3. Enter: `admin@test.com` / `password123`
4. You should be redirected to `/dashboard`

---

## 🎯 Remaining Tasks

The following pages still need to be updated to use the Supabase hooks and services:

1. **Dashboard.tsx** - Display real statistics from database
2. **Students.tsx** - Implement student CRUD operations
3. **Teachers.tsx** - Implement teacher CRUD operations
4. **Classes.tsx** - Implement class CRUD operations
5. **Attendance.tsx** - Implement attendance marking
6. **Timetable.tsx** - Implement timetable management
7. **Announcements.tsx** - Implement announcements with real-time
8. **Settings.tsx** - Implement profile management

### How to Use in Pages

Example for Students page:

```tsx
import {
  useStudents,
  useCreateStudent,
  useDeleteStudent,
} from "@/hooks/useStudents";

function Students() {
  const { data: students, isLoading } = useStudents();
  const createStudent = useCreateStudent();
  const deleteStudent = useDeleteStudent();

  // Use the data and mutations in your component
}
```

---

## 📚 Available Hooks

### Students

- `useStudents()` - Get all students
- `useStudent(id)` - Get single student
- `useStudentsByClass(classId)` - Get students by class
- `useCreateStudent()` - Create new student
- `useUpdateStudent()` - Update student
- `useDeleteStudent()` - Delete student

### Teachers

- `useTeachers()` - Get all teachers
- `useTeacher(id)` - Get single teacher
- `useCreateTeacher()` - Create new teacher
- `useUpdateTeacher()` - Update teacher
- `useDeleteTeacher()` - Delete teacher

### Classes

- `useClasses()` - Get all classes
- `useClass(id)` - Get single class
- `useCreateClass()` - Create new class
- `useUpdateClass()` - Update class
- `useDeleteClass()` - Delete class

### Attendance

- `useAttendance()` - Get all attendance
- `useAttendanceByDate(date)` - Get by date
- `useAttendanceByClassAndDate(classId, date)` - Get by class and date
- `useMarkAttendance()` - Mark single attendance
- `useBulkMarkAttendance()` - Mark multiple attendance

### Timetable

- `useTimetable()` - Get all timetable
- `useTimetableByClass(classId)` - Get by class
- `useTimetableByTeacher(teacherId)` - Get by teacher
- `useCreateTimetable()` - Create entry
- `useUpdateTimetable()` - Update entry
- `useDeleteTimetable()` - Delete entry

### Announcements

- `useAnnouncements()` - Get all
- `usePublishedAnnouncements()` - Get published only
- `useAnnouncementsByRole(role)` - Get by role
- `useCreateAnnouncement()` - Create announcement
- `useUpdateAnnouncement()` - Update announcement
- `useDeleteAnnouncement()` - Delete announcement
- `useAnnouncementsRealtime()` - Subscribe to real-time updates

---

## 🔒 Authentication

### Available from `useAuth()` hook:

- `user` - Current user profile
- `supabaseUser` - Supabase auth user
- `session` - Current session
- `loading` - Loading state
- `signIn(email, password)` - Sign in
- `signUp(email, password, fullName, role)` - Sign up
- `signOut()` - Sign out
- `updateProfile(updates)` - Update profile

---

## 🛠️ Troubleshooting

### "Missing Supabase environment variables"

- Make sure `.env.local` exists in project root
- Restart dev server after adding env variables

### "JWT expired" or auth errors

- Check if Supabase project is active
- Verify your credentials are correct

### Tables not found

- Make sure you ran all SQL commands in Supabase dashboard
- Check table names match the schema

### RLS errors

- Verify Row Level Security policies are created
- Check user role is set correctly in users table

---

## 📦 Files Created

```
src/
├── contexts/
│   └── AuthContext.tsx
├── components/
│   └── ProtectedRoute.tsx
├── lib/
│   ├── supabase.ts
│   ├── database.types.ts
│   └── services/
│       ├── studentsService.ts
│       ├── teachersService.ts
│       ├── classesService.ts
│       ├── attendanceService.ts
│       ├── timetableService.ts
│       ├── announcementsService.ts
│       └── storageService.ts
├── hooks/
│   ├── useStudents.ts
│   ├── useTeachers.ts
│   ├── useClasses.ts
│   ├── useAttendance.ts
│   ├── useTimetable.ts
│   └── useAnnouncements.ts
.env.local
```

---

## ✨ You're all set!

The Supabase backend is now fully integrated. Complete the database setup in Supabase dashboard, then start updating your page components to use the provided hooks and services.
