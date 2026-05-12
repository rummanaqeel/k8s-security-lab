# Troubleshooting Guide — Kubernetes Security Lab

This document covers every common issue encountered during this lab with exact fixes.

---

## EC2 / SSH Issues

### Issue: Permission denied (publickey) when SSH-ing
**Cause:** Wrong key file, wrong username, or wrong permissions on .pem file
**Fix:**
```bash
# Set correct permissions on key file
chmod 400 /path/to/k8s-security-lab.pem

# Make sure you're using 'ubuntu' not 'root' or 'ec2-user'
ssh -i /path/to/k8s-security-lab.pem ubuntu@YOUR_PUBLIC_IP
```
If still failing — check your security group has SSH (port 22) open to your current IP. Your IP may have changed if you're on a dynamic IP.

---

### Issue: Connection timed out when trying to SSH
**Cause:** Security group not allowing your IP, or instance not fully started
**Fix:**
- Wait 2 more minutes and try again
- Go to EC2 Console → Security Groups → your security group → Edit inbound rules → change SSH source to "My IP" (AWS will auto-detect your current IP)

---

### Issue: Instance shows "running" but status checks showing 1/2
**Cause:** Instance still initialising
**Fix:** Wait 3-5 more minutes. Status checks take time after instance shows running. Do not reboot.

---

## K3s Installation Issues

### Issue: curl command hangs or fails during K3s install
**Cause:** Network connectivity issue or DNS resolution failure
**Fix:**
```bash
# Test connectivity first
curl -I https://get.k3s.io

# If that fails test DNS
nslookup get.k3s.io

# If DNS fails restart networking
sudo systemctl restart systemd-resolved
```

---

### Issue: kubectl get nodes shows "NotReady" after K3s install
**Cause:** K3s still starting up or CNI not ready
**Fix:**
```bash
# Wait 60 seconds then check again
sleep 60 && sudo kubectl get nodes

# Check K3s logs for errors
sudo journalctl -u k3s --since "5 minutes ago" | tail -50
```
If still NotReady after 3 minutes:
```bash
sudo systemctl restart k3s
sleep 30
sudo kubectl get nodes
```

---

### Issue: kubectl command not found after installation
**Cause:** K3s installs kubectl but PATH not updated in current session
**Fix:**
```bash
# K3s puts kubectl at /usr/local/bin/kubectl
export PATH=$PATH:/usr/local/bin
# Or use k3s kubectl directly
sudo k3s kubectl get nodes
```

---

### Issue: Error "The connection to the server was refused" when running kubectl
**Cause:** KUBECONFIG not set correctly or K3s API server not running
**Fix:**
```bash
# Check if K3s is running
sudo systemctl status k3s

# Reset kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config
kubectl get nodes
```

---

## Pod / Deployment Issues

### Issue: Pod stays in "Pending" state
**Cause:** Insufficient resources or image pull issue
**Fix:**
```bash
# Check what's happening with the pod
kubectl describe pod [POD_NAME] -n production

# Look at Events section at the bottom of output
# Common causes shown there: ImagePullBackOff, Insufficient CPU/memory
```

If it's resource related on t2.micro — switch to t2.medium as specified in the guide.

---

### Issue: Pod shows "ImagePullBackOff" or "ErrImagePull"
**Cause:** Cannot pull image from Docker Hub (network issue or rate limit)
**Fix:**
```bash
# Check pod events
kubectl describe pod [POD_NAME] -n [NAMESPACE]

# Try pulling image manually first
sudo crictl pull nginx:latest

# If Docker Hub rate limited, wait 15 minutes or use a different image tag
```

---

### Issue: kubectl exec into pod fails with "error: unable to upgrade connection"
**Cause:** Network/firewall issue between kubectl client and K3s
**Fix:**
```bash
# This happens when running kubectl on the same node - use this flag
kubectl exec -it [POD_NAME] -n production -- /bin/bash --request-timeout=60s

# Alternative - check if exec is blocked by network policy
# Temporarily remove network policies to test
kubectl delete networkpolicy default-deny-all -n production
# Then retry exec, then reapply policy
kubectl apply -f ~/k8s-security-lab/manifests/network-policies/default-deny-all.yaml
```

---

## Falco Issues

### Issue: Falco fails to install — kernel headers not found
**Cause:** Kernel headers package not available for current kernel version
**Fix:**
```bash
# Check your kernel version
uname -r

# Install matching headers
sudo apt-get install -y linux-headers-$(uname -r)

# If not found in repos, try
sudo apt-get install -y linux-headers-generic
```

---

### Issue: Falco service fails to start — driver not loaded
**Cause:** eBPF or kernel module not loading correctly
**Fix:**
```bash
# Check the exact error
sudo journalctl -u falco -n 50

# Try forcing driver rebuild
sudo falco-driver-loader

# If that fails try the legacy kernel module instead
sudo falco-driver-loader module
sudo systemctl restart falco
```

---

### Issue: Falco running but no alerts appearing
**Cause:** Events not being triggered or Falco watching wrong log target
**Fix:**
```bash
# Verify Falco is actually watching
sudo journalctl -fu falco

# Make sure you're triggering events in a RUNNING pod (not completed/errored)
kubectl get pods -n production

# Trigger a clear event - this always works
kubectl exec -it $(kubectl get pod -n production -l app=nginx-app -o jsonpath='{.items[0].metadata.name}') -n production -- whoami
```

---

### Issue: "No pods found" error when running kubectl exec command with jsonpath
**Cause:** Pod label selector not matching, or pod not running
**Fix:**
```bash
# Check actual pod labels
kubectl get pods -n production --show-labels

# Get pod name manually then use it directly
kubectl get pods -n production
# Copy the pod name from output then
kubectl exec -it [EXACT_POD_NAME] -n production -- /bin/bash
```

---

## Trivy Issues

### Issue: Trivy scan takes very long or hangs
**Cause:** Downloading vulnerability database on first run (normal — takes 3-5 minutes)
**Fix:** Wait. First run always downloads the DB. Subsequent scans are much faster.

---

### Issue: Trivy shows "FATAL" error about DB download
**Cause:** Network issue or GitHub rate limiting
**Fix:**
```bash
# Clear Trivy cache and retry
rm -rf ~/.cache/trivy
trivy image --download-db-only
trivy image nginx:latest
```

---

## kube-bench Issues

### Issue: kube-bench binary not executing
**Cause:** Wrong permissions or wrong architecture
**Fix:**
```bash
# Check architecture
uname -m
# Should show x86_64

# Fix permissions
chmod +x ./kube-bench

# Verify download not corrupted
ls -lh kube-bench
# Should be several MB not a few KB
```

---

### Issue: kube-bench shows "could not find config file" errors
**Cause:** K3s config paths differ from standard Kubernetes
**Fix:**
```bash
# Run kube-bench with K3s specific config
sudo ./kube-bench --config-dir cfg --config cfg/config.yaml run --targets node
```

---

## GitHub Push Issues

### Issue: git push rejected — authentication failed
**Cause:** GitHub no longer accepts password authentication
**Fix:**
```bash
# Generate a Personal Access Token on GitHub:
# GitHub.com → Settings → Developer Settings → Personal Access Tokens → Generate new token
# Select: repo scope
# Copy the token

# Then when prompted for password during git push — paste the TOKEN not your password

# Or set up token permanently
git remote set-url origin https://YOUR_TOKEN@github.com/rummanaqeel/REPO_NAME.git
```

---

### Issue: Large files rejected by GitHub
**Cause:** GitHub has 100MB file size limit
**Fix:**
```bash
# Check file sizes before pushing
du -sh ~/k8s-security-lab/*

# If scan results are too large, create a summary instead
head -200 ~/k8s-security-lab/scan-results/nginx-scan.txt > ~/k8s-security-lab/scan-results/nginx-scan-summary.txt
```

---

## General Kubernetes Debugging Commands

These are your go-to commands when something is not working:

```bash
# Check all resources in a namespace
kubectl get all -n production

# Get detailed info about a resource
kubectl describe [resource-type] [resource-name] -n [namespace]

# Check logs of a pod
kubectl logs [pod-name] -n [namespace]

# Check previous logs if pod crashed
kubectl logs [pod-name] -n [namespace] --previous

# Get events in a namespace (shows what went wrong)
kubectl get events -n production --sort-by='.lastTimestamp'

# Check K3s system logs
sudo journalctl -u k3s --since "10 minutes ago"

# Restart K3s if cluster is behaving strangely
sudo systemctl restart k3s
sleep 30
kubectl get nodes
```

---

## Cost Protection Reminder

When you are completely done with this lab, terminate your EC2 instance to avoid charges:

1. Go to EC2 Console
2. Select your instance
3. Actions → Instance State → **Terminate** (not Stop — Terminate)
4. Confirm termination

Running t2.medium costs $0.0464/hour. Forgetting to terminate = unexpected charges.
