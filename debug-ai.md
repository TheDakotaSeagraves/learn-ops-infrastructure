# learn-ops-api: Invisible Save Debugging

Branch: `debug/invisible-save-ai`

## 1. The bug report

A user updates their Slack handle from the client, gets what looks like a successful response, but on refresh the old value is back. This points at one of two things: the response is a false positive (something upstream reports success without writing to the DB), or the write itself silently fails.

## 2. Walkthrough of `StudentViewSet.update` (`LearningAPI/views/student_view.py`)

Before the fix, the method looked like this:

```python
def update(self, request, pk=None):
    try:
        student = NssUser.objects.get(pk=pk)

        if request.auth.user == student.user or request.auth.user.is_staff:
            if "slack_handle" in request.data:
                student.slack_handle = request.data["slack_handle"]
            if "gitub_handle" in request.data:
                student.gitub_handle = request.data["gitub_handle"]

            return Response(None, status=status.HTTP_204_NO_CONTENT)
        else:
            return Response(None, status=status.HTTP_401_UNAUTHORIZED)

    except NssUser.DoesNotExist:
        return Response(None, status=status.HTTP_404_NOT_FOUND)

    except Exception as ex:
        return HttpResponseServerError(ex)
```

Step by step:

1. Fetch the `NssUser` row by primary key.
2. Authorize: the caller must be the student themselves, or a staff user.
3. If `slack_handle` is present in the PUT body, assign it to the in-memory model instance.
4. If `gitub_handle` (note: misspelled — should be `github_handle`) is present, assign it too.
5. Return `204 No Content`.

## 3. Root cause

Step 3 assigns `student.slack_handle = request.data["slack_handle"]` but never calls `student.save()`. In Django's ORM, fetching a row via `.get()` loads its data into a plain Python object. Setting an attribute on that object is ordinary Python attribute assignment — it does not talk to the database. Django only emits an `UPDATE` statement when `.save()` (or a bulk `.update()` on a queryset) is explicitly called.

Because the method still returns `204 No Content` after the assignment, the client sees a "successful" response even though the row was never written. On the next page load, the frontend does a fresh `GET`, which reads from the database and returns the untouched original value — hence the "invisible save."

This is a "false success" bug rather than a crash: no exception is raised, so the `except` blocks never fire and nothing in the logs would flag it.

## 4. Secondary issue found (not the reported bug, but adjacent)

`"gitub_handle"` on the line below is a typo for `"github_handle"`. Even with `.save()` added, a client sending `github_handle` (the correct field name) would silently have that update ignored, since the code checks for the misspelled key. This wasn't in scope for the Slack handle fix, but is worth flagging separately since it's the same failure pattern (silent no-op) on a different field.

## 5. Reproducing with a test first

Before changing the view, we wrote `test_patch_slack_handle_persists_to_database` in `LearningAPI/tests/test_student_update.py` to pin down the bug with an assertion rather than manual poking:

```python
response = client.put(
    f"/students/{nss_user.id}",
    {"slack_handle": "@newhandle"},
    format="json"
)

self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

nss_user.refresh_from_db()
self.assertEqual(nss_user.slack_handle, "@newhandle")
```

Run against the unfixed view, this failed exactly as predicted:

```
AssertionError: '@old_handle' != '@newhandle'
```

The response status was 204 (looked successful), but `refresh_from_db()` — which re-queries the database for the row's current state — showed the old value was still there. This is the same mechanism the user experienced via page refresh, reproduced deterministically in a test.

## 6. The fix

One line, added after both attribute assignments and before the `return`, inside the authorized branch:

```python
if "gitub_handle" in request.data:
    student.gitub_handle = request.data["gitub_handle"]

student.save()

return Response(None, status=status.HTTP_204_NO_CONTENT)
```

Placement matters: it has to come after the assignments (so whichever fields were actually set get persisted) and before the response is returned (so the DB write happens before the client is told it succeeded).

## 7. Verification

- `test_patch_slack_handle_persists_to_database` passes after the fix.
- Full suite (`docker compose exec api pytest`) — 20/20 tests passed, no regressions introduced by adding the `.save()` call.

## 8. Takeaway

Django model instances and database rows are two different things connected only by explicit `.save()`/`.refresh_from_db()` calls. A method that mutates attributes and returns a success status without saving will look correct in manual testing (the in-memory object reflects the new value for the rest of the request) but silently fails to persist — exactly the class of bug that's easy to miss without a test that re-fetches from the database rather than trusting the in-memory object.
