# GameHub — Understanding the system

10 questions to test your understanding of the data flow and architecture.
Work through them in order: read the code first, then run the app, then try to break things.

---

## How to investigate

You will need three things:

**1. Read the source code**
Start with `models.py` (the schema), then `seed.py` (the data), then `app.py` (the logic).
Many questions are answered entirely by reading carefully.

**2. Run the app and interact with it**
Use the UI at `http://localhost:5000` or send requests with curl or Postman.
Observe what actually happens — don't just reason about it.

```bash
# Example: log an activity for nova (id=1) on Hollow Knight (id=1)
curl -X POST http://localhost:5000/activities \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "game_id": 1, "action": "started"}'
```

**3. Query the database directly**
Open `gamehub.db` with a SQLite tool and inspect the actual rows.

```bash
sqlite3 gamehub.db
.tables
SELECT COUNT(*) FROM notifications;
SELECT * FROM notifications WHERE user_id = 1;
```

Or use a GUI: **DB Browser for SQLite** (free, recommended).

---

## Suggested approach

| Phase | Questions | What you are doing |
|-------|-----------|-------------------|
| Read first | 1, 4, 8, 10 | Understand the code before touching anything |
| Then run it | 3, 6, 9    | Observe actual behaviour                     |
| Then break it | 2, 5, 7  | Try things, hit walls, reason about why      |

---

## Questions

**1.** When a user logs a new activity, how many database tables are written to?
List them and explain why each one is affected.
```
It depends on the activity, but typically the activity is written to the activities table and a notification is written to the notifications table. The activity stores information such USER_ID and GAME_ID. Then, the notification stores a message, USER_ID, and a reference to the activities table through the activity ID.
```

---

**2.** You call `DELETE FROM users WHERE id = 3` directly in SQLite.
What happens, and why? What would you need to do instead?
```
After executing this query directly in SQLite, the message 'Query executed successfully but returned no data.' is returned. After looking at the database, we can confirm that the user is actually deleted. However, all of the user's history is still saved in the other tables that reference it. In order to fix this, we should utilize a cascade to remove the references to this user.
```

---

**3.** User `nova` changes her username to `nova_2`.
She then checks her friends' notification feeds.
What do they see — the old name or the new one? Why?
```
She will see the new name, as the username isn't directly stored in the notifications table. Instead, the USER ID is stored to reference the users table. Subsequently, since the username has been updated in users table, when the notifications table is called, it will grab the updated username.
```

---

**4.** Trace the full journey of a `POST /activities` request.
Starting from the HTTP call, list every operation that happens before the response is returned.
```
After we send the POST request, the activity is inserted into the activities table, the ID is fetched by a SELECT query, the username and game are fetched by a reference to their IDs, and a notification is created for each friend in the notifications table. Finally, the JSON body is returned with a '201 OK' message: {
    "action": "started",
    "created_at": "2026-05-13T09:49:48Z",
    "game_id": 1,
    "id": 16,
    "user_id": 1
}
```

---

**5.** `pixel_queen` opts out of activity tracking.
A teammate adds an `opted_out` boolean column to the `users` table and updates the `POST /activities` API route to check it.
Is the feature fully implemented? What did they miss?
```
The feature is not yet fully implemented. The information will still be added to the users, but, in order to also add the opted_out value, the HTML form route would have to be updated. Furthermore, the lack of functionality of this feature in not GDPR compliant: big time no-no (as we've learned).
```

---

**6.** How many rows are created in the database when `nova` logs one activity, given the current seed data?
Show your working.
```
Only one row is created in the activities table, and three rows (for friends with user IDs 2, 3, and 5) are created in the notifications table. I'm not sure what you mean by 'Show your working.', but I see one activity and three notifications created in my SQLite IntelliView Extension.
```

---

**7.** You need to delete `maya_r`.
In what order must you delete rows across the tables, and why does the order matter?
```
The order matters because several tables have foreign keys that reference other tables, and the foreign keys are enforced. You must first delete rows from activities, then you can delete friends, games, and notifications in any order. Then, after all of the references have been deleted, you can delete maya_r from the users table.
```

---

**8.** The `notifications` table has a foreign key pointing to `activities`.
What happens if you try to delete an activity that has notifications attached to it?
```
You cannot delete an activity that has notifications attached to it: again, foreign key constraints are enforced. You'll get an error.
```

---

**9.** A bug is found in the game catalog — wrong genre for one game.
You fix it and restart the app to ship the change.
What else just went down, and for how long?
```
While you restart the app to ship the change, everything goes down: you cannot access the app functionality if the app isn't running. It will be down the period in which the server is restarting, so however long it takes for the. server to go from off to on during the restart. We've actually learned ways around this in other courses (specifically cyber security), but we can even allow for hot fixes with things like Blue and Green servers.
```

---

**10.** A teammate says: *"let's just move the notification logic into its own function in `app.py`"*.
Does that solve the problem described in Task 4?
What is the actual architectural issue?
```
If we don't change anything about the functionality, and we just move it into its own function, then nothing will happen. We're just moving the location of the code, not actaully changing the logic of it to be properly functional. The architectural issue is that the notifications and activities routes are groupped together.
```
