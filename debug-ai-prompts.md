# learn-ops-api: Debugging Session AI Prompts

## 1. Environment setup
"git checkout main
git pull
git checkout -b debug/invisible-save-ai"

## 2. Start the environment
"run the docker containers"

## 3. Initial bug report and walkthrough request
"Read LearningAPI/views/student_view.py and focus on the `update` method.

A user reports that updating their Slack handle appears to succeed, but the change is gone after a page refresh. Walk me through what the `update` method does step by step, and identify what would cause a change to `slack_handle` to not persist to the database."

## 4. Drilling into the root cause
"Which specific line is missing, and what Django concept explains why the change does not reach the database without it?"

## 5. Open the editor
"code ."

## 6. Write a failing test that proves the bug
"Look at LearningAPI/tests/test_student_update.py. A test class is already set up with a user, a token, and an NssUser with slack_handle='@old_handle'. The method `test_patch_slack_handle_persists_to_database` currently contains only `fail("Not Implemented yet")`.

Write the body of that method so it:
1. Makes a PUT request to `/students/{pk}/` with a new slack_handle value
2. Asserts the response status is 204
3. Re-fetches the NssUser from the database
4. Asserts the re-fetched record's slack_handle matches the value sent"

## 7. Confirm the test fails against the buggy code
"docker compose exec api pytest LearningAPI/tests/test_student_update.py -v"

## 8. Pin down the exact fix location
"What is the single line that fixes the bug in the `update` method, and where exactly in the method does it go?"

## 9. Re-run before applying the fix (sanity check)
"docker compose exec api pytest LearningAPI/tests/test_student_update.py -v"

## 10. Apply the fix
"Add the student.save() fix"

## 11. Confirm no regressions
"run the full test suite"

## 12. Document the exercise
"in learn-ops-infrastructure make these two files "debug-ai.md" "debug-ai-prompts.md" and add our explonations and findings with every prompt you sent to Claude during this exercise."
