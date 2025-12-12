# Run Docker Containers with Unprivileged Users

When running Docker containers, it's a security best practice to avoid running applications as the root user inside the container. An attacker who gains access to a shell inside a container running as root has elevated privileges within the container, which increases the attack surface and potential impact if combined with container escape vulnerabilities. To mitigate this risk, you can configure Docker containers to run as unprivileged users.

> [!IMPORTANT]
> Not all containers can be easily configured to run as a different user. Some applications require root privileges to function correctly. For example, applications that bind to ports below 1024 require root privileges. Always refer to the documentation of the specific Docker image you are using to determine if running as a non-root user is supported.

## Steps to Run Docker Containers as Unprivileged Users

First, create a dedicated user on the host system which will be used to run the Docker container. It is recommended to create a new user for every container, with a group named the same as the user.

> [!IMPORTANT]
> This guide assumes you are familiar with SSH access to your UGREEN NASync device and have basic knowledge of Docker commands.

1. Create a new user on your UGREEN NASync device that will be used to run the Docker container.

   For example, to create a user named `nginx` in group `nginx`:

   ```bash
   sudo useradd --system --user-group --shell /usr/sbin/nologin nginx
   ```

2. Find out the UID and GID of the newly created user and group:

   ```bash
   $ id nginx
   uid=995(nginx) gid=991(nginx) groups=991(nginx)
   ```

3. When running the Docker container, use the `--user` flag to specify the UID and GID of the unprivileged user:

   ```bash
    # Replace with your UID:GID
   docker run -d --user 995:991 \
       --name my_nginx_container nginx
   ```

   When using `docker compose`, you can specify the user in the compose file:

   ```yaml
   services:
     web:
       image: nginx:stable
       user: "995:991" # Replace with your UID:GID
       ports:
         - "8080:8080"
       volumes:
         - ./nginx.conf:/etc/nginx/nginx.conf:ro
         - ./html:/usr/share/nginx/html:ro
   ```

4. Ensure that any volumes or bind mounts used by the container have the appropriate permissions set for the unprivileged user. You may need to adjust permissions using the `ugacltool` command to set the correct ACLs.

   > [!IMPORTANT]
   > Linux standard `chown` and `chmod` commands don't work well on UGREEN NASync due to its custom ACL system.
   > Use the `ugacltool` command to set the correct permissions for the unprivileged user.

   For example, to set the correct permissions for the `nginx` user on the mounted volumes, you can use:

   ```bash
    # Grant read + execute (no write) to group "nginx" for files/dirs
    ugacltool add ./nginx.conf group:nginx:allow:r-x---a-R-c--:-fd-
    ugacltool add ./html group:nginx:allow:r-x---a-R-c--:-fd-
    # To allow write access (e.g., for apps that need to write), use:
    # ugacltool add ./html group:nginx:allow:rwxpdDaARWc--:-fd-
   ```

   If you use a different username, substitute both the `user: "UID:GID"` and the ACL `group:<name>` accordingly. Ensure the container's process runs with the same UID/GID as the permissions granted on the host-mounted paths.

5. (Container specific) Depending on the application, you might need to adjust additional settings inside the container to ensure it runs correctly as the unprivileged user.

   For example, nginx by default listens on port 80, which requires root privileges. You can change the listening port to a higher number (e.g., 8080) in the nginx configuration file.

   ```diff
   server {
   -     listen       80;
   +     listen       8080;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
   }
   ```

6. Validate the configuration:

   ```bash
   # Confirm the UID:GID mapping
   id nginx
   # Inspect effective user inside the running container
   docker exec -it my_nginx_container id
   # Test file access
   docker exec -it my_nginx_container ls -l /usr/share/nginx/html
   ```

   If the IDs inside the container differ from the host user, adjust the `user:` field to match host UID/GID, or update ACLs to the correct group.

7. Recreate and start the Docker container to apply the changes.

## ACL Tool Reference

### Cheat Sheet

Grant read-only access to group `nginx` for a PATH (file or directory):

```bash
ugacltool add PATH group:nginx:allow:r-x---a-R-c--:-fd-
```

Grant read+write access to group `nginx` for a PATH (file or directory):

```bash
ugacltool add PATH group:nginx:allow:rwxpdDaARWc--:-fd-
```

Get current ACL entries for a PATH:

```bash
ugacltool get PATH
```

Remove one ACL entry by index:

```bash
ugacltool del_one PATH INDEX
```

> [!TIP]
> The `del_one` command only works with level 0 ACLs.  
> Use `ugacltool get PATH` to find the index number of the ACL entry you want to remove.

### All Options

```text
usage: ugacltool <command> [options] <path>

    ugacltool add PATH [ACL Entry]
        add ug ace into file
    ugacltool addace PATH [ACL Entry]
        add ug ACEs with uid/gid into file
    ugacltool replace PATH [ACL Entry Index] [ACL Entry]
        replace specified ace by index number
    ugacltool set_eadir_acl PATH
        set_eadir_acl: set ACL for EA dir
    ugacltool get PATH
        get: get ug acl of file
    ugacltool getace PATH
        getace: get ug ACEs with uid/gid of file
    ugacltool get_perm PATH USRNAME
        get_perm: extract windows permission from acl or linux permission
    ugacltool get_perms USRNAME PATHS
        get_perms: extract windows permissions from acl or linux permission
    ugacltool set_archive PATH [ACL Archive Option]
        set_archive: set ACL archive bit
    ugacltool get_archive PATH
        get_archive: get ACL archive bit
    ugacltool del_archive PATH [ACL Archive Option]
        del_archive: delete ACL archive bit
    ugacltool del_one PATH [ACL Entry Index]
        delete one ug acl of file
    ugacltool del_all PATH
        delete all ug acl of file
    ugacltool copy PATH_SRC PATH_DST
        copy: copy ACL from source to destination, only works when ACL exists
    ugacltool check PATH [ACL Perm]
        check: check acl permission of file
    ugacltool stat PATH
        stat: get stat/archive bit
    ugacltool lstat PATH
        lstat: get stat/archive bit
    ugacltool fstat PATH
        fstat: get stat/archive bit
    ugacltool utime PATH
        utime: set current time into file
    ugacltool enforce_inherit PATH
        enforce_inherit: enforce ACL inheritance

OPTIONS
         ACL Entry Index: >= 0
         ACL Option: [inherit|single]
         ACL Archive Option: is_inherit,is_read_only,is_owner_group,has_ACL,is_support_ACL
         ACL Entry: [user|group|owner|everyone|authenticated_user|system]:name:[allow|deny]:permissions:inherit mode

                         Example: [0] user:root:allow:rwx-d---RWc--:fd--
                         Example: [1] owner:*:allow:rwx-d---RWc--:fd--
                         Fields
                                         name: user/group name
                                         ACL Perm: rwxpdDaARWcCo
                                                          r: (r)ead data
                                                          w: (w)rite data (create file)
                                                          x: e(x)ecute
                                                          p: a(p)pend data (create dir)
                                                          d: (d)elete
                                                          D: (D)elete child (only for dir)
                                                          a: read (a)ttribute (For SMB read-only/hidden/archive/system)
                                                          A: write (A)ttribute
                                                          R: (R)ead xattr
                                                          W: (W)rite xattr
                                                          c: read a(c)l
                                                          C: write a(c)l
                                                          o: get (o)wner ship

                                         inherit mode: fdin
                                                          f: (f)ile inherited
                                                          d: (d)irectory inherited
                                                          i: (i)nherit only
                                                          n: (n)o propagate
```
