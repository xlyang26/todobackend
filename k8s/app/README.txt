The MYSQL_PASSWORD must be set as an environment variable. The deployment.yaml-orig and migrate.yaml-orig don't have the variable set in envrionment. Thus, both the todobackend application and the migiate job failed, basically with error below:

   mysql.connector.errors.ProgrammingError: 1045 (28000): Access denied for user 'todo'@'10.1.2.214' (using password: YES)

(As seen on the browser for app, and in the migrate-job log, via 'kubectl logs migrate-pod' command)

The fix is to set up the MYSQL_PASSWORD in both pods as shell envrionment variable, see:
   https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/
   section: "Create a Pod that has access to the secret data through environment variables"

Steps:

'kcl': kubectl desktop cluster

$ kcl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-d5rtr   kubernetes.io/service-account-token   3         3d
todobackend-secret    Opaque

1. Add to k8s/app/deployment.yaml and k8s/app/migrate.yaml (in env, after MYSQL_USER):
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: todobackend-secret
              key: MYSQL_PASSWORD

2. Redeploy app and migrate-job:
   $ kcl apply -f k8s/app

Verification:
1. In browser: http://localhost:8001/todos, the app is completely working.
2. $ kcl get pods
NAME                              READY     STATUS      RESTARTS   AGE
todobackend-55587fd6b9-jzjv2      1/1       Running     0          22m
todobackend-55587fd6b9-nfbz9      1/1       Running     0          22m
todobackend-db-5466fc8c7b-bbggq   1/1       Running     0          16h
todobackend-migrate-svxdm         0/1       Completed   0          20m
3. $ kcl logs todobackend-migrate-svxdm
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, todo
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying sessions.0001_initial... OK
  Applying todo.0001_initial... OK
   
