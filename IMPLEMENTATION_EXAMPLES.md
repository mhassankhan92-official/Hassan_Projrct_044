# Quick Implementation Examples

## Example 1: Students Page with Full CRUD

```tsx
import { useState } from "react";
import {
  useStudents,
  useCreateStudent,
  useDeleteStudent,
} from "@/hooks/useStudents";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import DashboardLayout from "@/components/DashboardLayout";

export default function Students() {
  const { data: students, isLoading, error } = useStudents();
  const createStudent = useCreateStudent();
  const deleteStudent = useDeleteStudent();
  const [isDialogOpen, setIsDialogOpen] = useState(false);

  if (isLoading) {
    return (
      <DashboardLayout>
        <div className="space-y-4">
          <Skeleton className="h-12 w-full" />
          <Skeleton className="h-12 w-full" />
          <Skeleton className="h-12 w-full" />
        </div>
      </DashboardLayout>
    );
  }

  if (error) {
    return (
      <DashboardLayout>
        <div className="text-center text-red-500">
          Error loading students: {error.message}
        </div>
      </DashboardLayout>
    );
  }

  const handleCreateStudent = async (formData: any) => {
    await createStudent.mutateAsync({
      userData: {
        email: formData.email,
        full_name: formData.fullName,
        role: "student",
      },
      studentData: {
        enrollment_number: formData.enrollmentNumber,
        date_of_birth: formData.dateOfBirth,
        gender: formData.gender,
        // ... other fields
      },
    });
    setIsDialogOpen(false);
  };

  const handleDeleteStudent = async (studentId: string) => {
    if (confirm("Are you sure you want to delete this student?")) {
      await deleteStudent.mutateAsync(studentId);
    }
  };

  return (
    <DashboardLayout>
      <div className="space-y-6">
        <div className="flex justify-between items-center">
          <h1 className="text-3xl font-bold">Students</h1>
          <Button onClick={() => setIsDialogOpen(true)}>Add Student</Button>
        </div>

        <div className="grid gap-4">
          {students?.map((student) => (
            <div key={student.id} className="border p-4 rounded-lg">
              <div className="flex justify-between items-center">
                <div>
                  <h3 className="font-semibold">{student.user.full_name}</h3>
                  <p className="text-sm text-muted-foreground">
                    {student.enrollment_number}
                  </p>
                </div>
                <Button
                  variant="destructive"
                  size="sm"
                  onClick={() => handleDeleteStudent(student.id)}
                >
                  Delete
                </Button>
              </div>
            </div>
          ))}
        </div>
      </div>
    </DashboardLayout>
  );
}
```

## Example 2: Attendance Marking

```tsx
import { useState } from "react";
import {
  useAttendanceByClassAndDate,
  useBulkMarkAttendance,
} from "@/hooks/useAttendance";
import { useStudentsByClass } from "@/hooks/useStudents";
import { useAuth } from "@/contexts/AuthContext";
import DashboardLayout from "@/components/DashboardLayout";
import { Button } from "@/components/ui/button";
import { Select } from "@/components/ui/select";

export default function Attendance() {
  const { user } = useAuth();
  const [selectedClass, setSelectedClass] = useState<string>("");
  const [selectedDate, setSelectedDate] = useState(
    new Date().toISOString().split("T")[0]
  );
  const [attendanceRecords, setAttendanceRecords] = useState<
    Record<string, string>
  >({});

  const { data: students } = useStudentsByClass(selectedClass);
  const { data: existingAttendance } = useAttendanceByClassAndDate(
    selectedClass,
    selectedDate
  );
  const bulkMark = useBulkMarkAttendance();

  const handleSubmit = async () => {
    if (!user) return;

    const records = students?.map((student) => ({
      student_id: student.id,
      class_id: selectedClass,
      date: selectedDate,
      status: attendanceRecords[student.id] || "absent",
      marked_by: user.id,
    }));

    if (records) {
      await bulkMark.mutateAsync(records);
    }
  };

  return (
    <DashboardLayout>
      <div className="space-y-6">
        <h1 className="text-3xl font-bold">Mark Attendance</h1>

        <div className="flex gap-4">
          <Select
            value={selectedClass}
            onValueChange={setSelectedClass}
            placeholder="Select Class"
          />
          <input
            type="date"
            value={selectedDate}
            onChange={(e) => setSelectedDate(e.target.value)}
            className="border rounded px-3 py-2"
          />
        </div>

        {students && (
          <div className="space-y-2">
            {students.map((student) => (
              <div
                key={student.id}
                className="flex items-center justify-between border p-3 rounded"
              >
                <span>{student.user.full_name}</span>
                <Select
                  value={attendanceRecords[student.id] || "present"}
                  onValueChange={(value) =>
                    setAttendanceRecords((prev) => ({
                      ...prev,
                      [student.id]: value,
                    }))
                  }
                >
                  <option value="present">Present</option>
                  <option value="absent">Absent</option>
                  <option value="late">Late</option>
                  <option value="excused">Excused</option>
                </Select>
              </div>
            ))}
          </div>
        )}

        <Button onClick={handleSubmit} disabled={!students?.length}>
          Submit Attendance
        </Button>
      </div>
    </DashboardLayout>
  );
}
```

## Example 3: Announcements with Real-time

```tsx
import {
  usePublishedAnnouncements,
  useAnnouncementsRealtime,
} from "@/hooks/useAnnouncements";
import { useAuth } from "@/contexts/AuthContext";
import DashboardLayout from "@/components/DashboardLayout";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";

export default function Announcements() {
  const { user } = useAuth();
  const { data: announcements, isLoading } = usePublishedAnnouncements();

  // Enable real-time updates
  useAnnouncementsRealtime();

  if (isLoading) {
    return <DashboardLayout>Loading announcements...</DashboardLayout>;
  }

  return (
    <DashboardLayout>
      <div className="space-y-6">
        <h1 className="text-3xl font-bold">Announcements</h1>

        <div className="space-y-4">
          {announcements?.map((announcement) => (
            <Card key={announcement.id}>
              <CardHeader>
                <div className="flex justify-between items-start">
                  <CardTitle>{announcement.title}</CardTitle>
                  {announcement.is_important && (
                    <Badge variant="destructive">Important</Badge>
                  )}
                </div>
              </CardHeader>
              <CardContent>
                <p className="text-muted-foreground">{announcement.content}</p>
                <div className="mt-4 text-sm text-muted-foreground">
                  By {announcement.author.full_name} •
                  {new Date(announcement.published_at!).toLocaleDateString()}
                </div>
              </CardContent>
            </Card>
          ))}
        </div>
      </div>
    </DashboardLayout>
  );
}
```

## Example 4: Dashboard with Statistics

```tsx
import { useStudents } from "@/hooks/useStudents";
import { useTeachers } from "@/hooks/useTeachers";
import { useClasses } from "@/hooks/useClasses";
import { useAttendanceByDate } from "@/hooks/useAttendance";
import DashboardLayout from "@/components/DashboardLayout";
import StatCard from "@/components/StatCard";
import { Users, GraduationCap, BookOpen, Calendar } from "lucide-react";

export default function Dashboard() {
  const { data: students } = useStudents();
  const { data: teachers } = useTeachers();
  const { data: classes } = useClasses();
  const { data: todayAttendance } = useAttendanceByDate(
    new Date().toISOString().split("T")[0]
  );

  const presentToday =
    todayAttendance?.filter((a) => a.status === "present").length || 0;
  const totalToday = todayAttendance?.length || 0;
  const attendanceRate =
    totalToday > 0 ? ((presentToday / totalToday) * 100).toFixed(1) : 0;

  return (
    <DashboardLayout>
      <div className="space-y-6">
        <h1 className="text-3xl font-bold">Dashboard</h1>

        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
          <StatCard
            title="Total Students"
            value={students?.length || 0}
            icon={Users}
            description="Enrolled students"
          />
          <StatCard
            title="Total Teachers"
            value={teachers?.length || 0}
            icon={GraduationCap}
            description="Active teachers"
          />
          <StatCard
            title="Total Classes"
            value={classes?.length || 0}
            icon={BookOpen}
            description="Active classes"
          />
          <StatCard
            title="Attendance Today"
            value={`${attendanceRate}%`}
            icon={Calendar}
            description={`${presentToday} out of ${totalToday}`}
          />
        </div>
      </div>
    </DashboardLayout>
  );
}
```

## Example 5: Settings/Profile Page

```tsx
import { useState } from "react";
import { useAuth } from "@/contexts/AuthContext";
import { storageService } from "@/lib/services/storageService";
import DashboardLayout from "@/components/DashboardLayout";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Avatar, AvatarImage, AvatarFallback } from "@/components/ui/avatar";

export default function Settings() {
  const { user, updateProfile, signOut } = useAuth();
  const [fullName, setFullName] = useState(user?.full_name || "");
  const [avatarFile, setAvatarFile] = useState<File | null>(null);
  const [isUploading, setIsUploading] = useState(false);

  const handleAvatarChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file || !user) return;

    setIsUploading(true);
    try {
      // Delete old avatar if exists
      if (user.avatar_url) {
        await storageService.deleteAvatar(user.avatar_url);
      }

      // Upload new avatar
      const avatarUrl = await storageService.uploadAvatar(user.id, file);

      // Update profile
      await updateProfile({ avatar_url: avatarUrl });
    } catch (error) {
      console.error("Error uploading avatar:", error);
    } finally {
      setIsUploading(false);
    }
  };

  const handleUpdateProfile = async () => {
    await updateProfile({ full_name: fullName });
  };

  return (
    <DashboardLayout>
      <div className="space-y-6 max-w-2xl">
        <h1 className="text-3xl font-bold">Settings</h1>

        <div className="space-y-4">
          <div>
            <Label>Profile Picture</Label>
            <div className="flex items-center gap-4 mt-2">
              <Avatar className="h-20 w-20">
                <AvatarImage src={user?.avatar_url || ""} />
                <AvatarFallback>{user?.full_name.charAt(0)}</AvatarFallback>
              </Avatar>
              <Input
                type="file"
                accept="image/*"
                onChange={handleAvatarChange}
                disabled={isUploading}
              />
            </div>
          </div>

          <div>
            <Label htmlFor="fullName">Full Name</Label>
            <Input
              id="fullName"
              value={fullName}
              onChange={(e) => setFullName(e.target.value)}
            />
          </div>

          <div>
            <Label>Email</Label>
            <Input value={user?.email} disabled />
          </div>

          <div>
            <Label>Role</Label>
            <Input value={user?.role} disabled />
          </div>

          <div className="flex gap-4">
            <Button onClick={handleUpdateProfile}>Save Changes</Button>
            <Button variant="destructive" onClick={signOut}>
              Sign Out
            </Button>
          </div>
        </div>
      </div>
    </DashboardLayout>
  );
}
```

## Key Points to Remember

1. **Always check loading and error states**
2. **Use optimistic updates for better UX**
3. **Show toast notifications for user feedback**
4. **Implement proper form validation**
5. **Use skeleton loaders while data is loading**
6. **Handle authentication checks with useAuth**
7. **Invalidate queries after mutations for fresh data**
8. **Use role-based access control where needed**

## Common Patterns

### Loading State

```tsx
if (isLoading) {
  return <Skeleton className="h-12 w-full" />;
}
```

### Error State

```tsx
if (error) {
  return <div>Error: {error.message}</div>;
}
```

### Mutation

```tsx
const mutation = useCreateStudent();

const handleSubmit = async (data) => {
  await mutation.mutateAsync(data);
  // Query automatically invalidated
};
```

### Real-time Subscription

```tsx
useAnnouncementsRealtime(); // Just call the hook
```
