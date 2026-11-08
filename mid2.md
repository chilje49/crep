```
snprintf(sql, sizeof(sql), "INSERT INTO equipment (name, location, status, voltage, check_date) VALUES ('%s', '%s', '%s', %.2f, '%s');", ...);
sqlite3_exec(db, sql, 0, 0, &err_msg);

```
