# Lab Postmortems: Containerised Web App Project (November 2026)

Environment: personal lab (WSL Ubuntu, Docker, GitHub). Both incidents happened during my Docker project, where I built: a custom Nginx image, a Docker Compose stack (web + MySQL 8), and an image published to Docker Hub (`esther09027/infracloud-web`). These are lab incidents, not production outages.

---

## Postmortem 1: Docker daemon unreachable in WSL

**Summary**
Docker commands stopped working with "failed to connect to the docker API at unix:///var/run/docker.sock". The Docker daemon was not running, so the app containers were unreachable until it was restarted.

**Impact**
- `docker ps` and `docker ps -a` failed
- The `InfraCloud` web container and the `infracloud-db` database container were unavailable
- No data or images were lost

**Detection**
Found manually when running `docker ps -a` after returning to the terminal.

**Timeline (order of events)**
1. Ran `docker ps -a`; got the `docker.sock: no such file or directory` error
2. Confirmed `docker ps` failed the same way
3. Concluded the daemon was stopped, not that Docker or my images were broken
4. Ran `sudo service docker start`, then `docker ps -a`; all containers and images were listed again
5. Ran `docker compose up -d` and `docker compose ps` to bring the stack back

**Root cause (suspected, not confirmed)**
The Docker service in WSL stops when the Ubuntu session closes or WSL restarts, and it does not start itself again. I did not capture logs to confirm exactly what stopped it.

**Resolution**
Restarted the service and brought the Compose stack back up. Images and containers had persisted on disk.

**What went well**
- The error message pointed straight at the daemon socket
- Data was safe because images and containers live on disk, not in the running process

**Follow-up actions**
- Check `sudo service docker status` first whenever Docker commands fail with a socket error
- Add a startup check to `~/.bashrc` so the service starts when a new terminal opens
- Capture `journalctl`/service logs next time to confirm the cause

---

## Postmortem 2: Repository changes not reaching GitHub

**Summary**
My local project and the GitHub repository got out of step. The remote contained an empty file named `screenshots`, while my local project had a folder with the same name. Afterwards the GitHub page also appeared not to show my latest push.

**Impact**
- Screenshots and README images could not be added cleanly to the repository
- Project documentation (screenshots) was delayed
- No work was lost, because everything was backed up before changes were made

**Detection**
Noticed when my screenshots folder would not sync with the remote repository.

**Timeline (order of events)**
1. Found that the remote's "Create screenshots" commit (`f74c8d0`), made through GitHub's web interface, had created an empty file named `screenshots`
2. That file blocked my local folder of the same name
3. Backed up the folder and README to `/tmp`
4. Ran `git reset --hard origin/main` to match the remote, then deleted the empty file
5. Recreated the `screenshots/` folder, restored the three images and README, and restored the Docker Hub profile link my local README had lost
6. Committed and pushed (`bcc55c9`)
7. GitHub in my browser still showed the old commit `f74c8d0`, so I ran `git ls-remote origin main` and confirmed the remote was at `bcc55c9`

**Root cause**
A file was created through GitHub's web interface with the same name as a folder in my local project, so the two histories conflicted. The "missing push" was only a stale browser view, not a failed push.

**Resolution**
Backed up first, reset to the remote, replaced the file with the folder, restored the README, then pushed and verified.

**What went well**
- Backed up everything before using `git reset --hard`
- Verified the push using the command line (`git ls-remote`) instead of trusting the browser

**Follow-up actions**
- Don't create files in the GitHub web interface when the same path exists locally
- Run `git fetch` and check status before starting work
- Verify pushes from the terminal first, and use a private window or hard refresh for the web view
