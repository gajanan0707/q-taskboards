# Code Review: Top 4 Critical Issues Ranked by Business Impact

This document details the top 4 critical issues identified in the TaskBoard backend (`backend/projects/` and `backend/users/`), ranked by their severity and business impact. No application code or seed data was modified.

---

## 1. Broken Object-Level Authorization (BOLA / IDOR) on Task Modification

- **File + Line:** `backend/projects/views.py:165-186` (`TaskDetailView.patch`)
- **Category:** Security
- **Severity:** High
- **Description:** `TaskDetailView.patch` fetches and mutates a task by its ID without checking whether the requesting user belongs to the task's project or holds an editing role (`admin` or `member`). As a result, read-only viewers and complete non-members can overwrite titles, descriptions, statuses, and assignees of any task across the entire system. This compromises tenant isolation and allows unauthorized cross-tenant data manipulation.
- **Recommended Fix:** Before updating task attributes, resolve the project membership for `request.user` via `_get_membership(request.user, str(task.project_id))` and enforce role permission using `_can_edit_tasks(membership.role)`. Return `HTTP_403_FORBIDDEN` if the membership does not exist or if the user is a viewer.

### Proof of Exploitation

#### Login Curl (Authenticate as Dev Sharma, a Read-Only Viewer on "Q3 Launch"):
```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"dev@example.com","password":"password123"}'
```

#### Exploit Curl (Modify task status and title as read-only viewer):
```bash
curl -X PATCH http://localhost:8000/api/tasks/1c262731-b193-4948-9e46-e20d29a89eda \
  -H "Authorization: Bearer <viewer_token>" \
  -H "Content-Type: application/json" \
  -d '{"status":"in_progress"}'
```

#### Real Server Response:
```json
{
  "task": {
    "id": "1c262731-b193-4948-9e46-e20d29a89eda",
    "project_id": "0c51d472-2a3a-425f-a6a8-f3d84d6b65cf",
    "title": "Finalize launch date with marketing",
    "description": "Detail for: Finalize launch date with marketing",
    "status": "in_progress",
    "assignee_id": "d9ed066f-3d48-457a-aa68-9f9acffc0bf6",
    "created_by_id": "d9ed066f-3d48-457a-aa68-9f9acffc0bf6",
    "position": 0,
    "created_at": "2026-10-15T07:05:23.024888Z",
    "updated_at": "2026-10-15T07:06:35.092987Z",
    "assignee": {
      "id": "d9ed066f-3d48-457a-aa68-9f9acffc0bf6",
      "email": "meera@taskboard.dev",
      "name": "Meera Iyer"
    }
  }
}
```
*(Status code: `200 OK`. The task was successfully modified by a viewer instead of being rejected with `403 Forbidden`.)*

---

## 2. Raw SQL Injection via Unescaped Query Parameter in Task Search

- **File + Line:** `backend/projects/views.py:110-120` (`TaskListCreateView.get`)
- **Category:** Security
- **Severity:** Critical
- **Description:** When the `q` search parameter is passed to `GET /api/projects/:id/tasks`, the raw string value is directly interpolated into a SQL string using an f-string instead of parameterized query arguments. Any authenticated user with access to a project can supply arbitrary SQL syntax via the `?q=` parameter. This exposes the PostgreSQL database to data exfiltration, authorization bypasses, and denial-of-service.
- **Recommended Fix:** Eliminate the manual raw SQL construction and execute search queries using Django's ORM: `Task.objects.filter(project_id=project_id).filter(Q(title__icontains=q) | Q(description__icontains=q))`. If raw SQL execution is required, use parameterized SQL placeholders (`cursor.execute(sql, [project_id, f'%{q}%', f'%{q}%'])`).

---

## 3. Missing Project Membership Validation on Task Assignees

- **File + Line:** `backend/projects/views.py:156, 180-181` (`TaskListCreateView.post`, `TaskDetailView.patch`)
- **Category:** Data Integrity
- **Severity:** Medium
- **Description:** Both task creation and update endpoints accept an arbitrary `assigneeId` and bind it directly to `task.assignee_id` without verifying that the referenced user is an active member of the project. Tasks can consequently be assigned to unrelated users from other projects or non-existent accounts. This corrupts project boundaries, breaks assignment dashboards, and leaks project task contexts across unrelated organizational teams.
- **Recommended Fix:** Add a validation step in both views ensuring that any non-null `assigneeId` exists in `Membership.objects.filter(project_id=project_id, user_id=assignee_id)`. If the user is not a member of the project, return an `HTTP_400_BAD_REQUEST` error with a descriptive message.

---

## 4. N+1 Database Query Overhead in Project Listing

- **File + Line:** `backend/projects/views.py:23-42` (`ProjectListCreateView.get`)
- **Category:** Performance
- **Severity:** Medium
- **Description:** Although `ProjectListCreateView.get` includes `prefetch_related('project__tasks')`, calling `p.tasks.count()` inside the response construction loop bypasses the prefetch cache and triggers an independent `SELECT COUNT(*)` query for each project. As users join more projects and task counts increase, this creates classic N+1 database roundtrips for every project dashboard view. Under concurrent user traffic, this creates unnecessary database connection pressure and degrades API response latency.
- **Recommended Fix:** Annotate the task counts directly on the initial Membership query using `django.db.models.Count('project__tasks')` (e.g. `Membership.objects.filter(user=request.user).annotate(task_count=Count('project__tasks'))`) and reference `m.task_count` directly in the loop.
