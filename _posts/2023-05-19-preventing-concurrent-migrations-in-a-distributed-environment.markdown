---
layout: post
title:  "Preventing Concurrent Migrations in a Distributed Environment"
date:   2023-05-19
---

<p class="intro"><span class="dropcap">I</span>n a distributed environment, where multiple apps share a single database, running migrations simultaneously can lead to conflicts and deployment failures. In this article, we'll explore a solution to prevent concurrent migrations and ensure smooth deployments. We'll discuss the issue, the approach used, and the code changes implemented to tackle this problem.</p>


### Problem

When running migrations in Rails, they are meant to run sequentially based on their timestamps. However, in a distributed environment like Heroku, we encountered an issue where multiple apps can trigger migrations simultaneously, causing conflicts and errors. This issue arose in our specific scenario where we have two staging apps, namely s-app and s-admin, both deployed on Heroku and sharing a single database.

Typically, Data Definition Language (DDL) statements execute instantly, so we hadn't encountered this problem before. However, due to recent changes, we introduced Data Manipulation Language (DML) statements in the migrations to update existing records, which introduced considerable processing time. It was during this process that we faced conflicts.

Let's delve into the technical details of what actually happens behind the scenes. Rails internally maintains a table called <span class="highlights">schema_migration</span>, which contains information about the migration status, whether it's <span class="highlights">up</span> or <span class="highlights">down</span>. When a migration starts running, it acquires an advisory lock. Once the migration is completed, the advisory lock is released, and the migration status changes from 'down' to 'up'.

In our case, with the inclusion of DML statements that prolonged the execution time, a migration acquired an advisory lock, causing a delay. During this delay, another app attempted to run the same migration but encountered the advisory lock already held by the previous migration in the other app. Rails threw an exception called <span class="highlights">'ActiveRecord::ConcurrentMigrationError'</span>, resulting in failed deployment.


### Approach:
To address this issue, we'll use a combination of advisory locks and environment variables to control the execution of migrations during deployment. The basic idea is to allow only one app to run migrations while the others skip them.

### Implementation:
Let's walk through the code changes needed to implement this solution.

Step 1: Create a release-tasks.sh Script
We'll start by creating a shell script called release-tasks.sh in the project's root directory. This script will handle the execution of migrations during the release phase.

bash
```
#!/bin/bash
set -e

if [ "$RUN_MIGRATIONS" = "true" ]; then
  echo "Running migrations"
  bundle exec rake db:migrate
else
  echo "Skipping migrations"
fi
```

The script checks the value of the RUN_MIGRATIONS environment variable. If it is set to "true," the script runs the migrations using bundle exec rake db:migrate. Otherwise, it skips the migrations.

Step 2: Grant Execution Permissions to the Script
To allow execution of the release-tasks.sh script, we need to grant it execution permissions. Run the following command:

bash
```
chmod +x release-tasks.sh
```
Step 3: Update the Procfile
In the project's Procfile, we need to modify the release command to use our custom script. Open the Procfile and update the line as follows:

```
release: ./release-tasks.sh
```

This change ensures that our release-tasks.sh script is executed during the release phase.

Step 4: Set Environment Variables for Each App
In Heroku, go to each app's settings and set the <span class="highlights">RUN_MIGRATIONS</span> environment variable. For the app that should run migrations, set <span class="highlights">RUN_MIGRATIONS</span> to "true". For the other apps that should skip migrations, either leave it unset or set it to any other value.

### Conclusion:
By implementing this solution, we have effectively prevented concurrent migrations in our distributed environment. By using advisory locks and environment variables, we ensure that only one app runs migrations while others skip them during deployment. This approach promotes a more stable and error-free deployment process.

Remember to test this solution thoroughly in a non-production environment before applying it to your live applications. It's crucial to consider the specific requirements and constraints of your own setup.

I hope this article helps you in handling concurrent migrations in your distributed environment. Feel free to reach out if you have any questions or need further assistance.

Happy coding!