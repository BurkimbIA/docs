# RunPod Setup Guide for Developers

This guide will help you set up RunPod and connect to your instances via SSH using VSCode.

## Prerequisites

- A terminal/command line (Linux, macOS, or Windows with Git Bash/WSL)
- VSCode installed on your local machine
- Basic familiarity with SSH and command line

## Step 1: Create a RunPod Account

1. Visit [runpod.io](https://www.runpod.io/)
1. Sign up using a **personal or work email** (avoid using burkimbia email)
1. Verify your email address

## Step 2: Generate and Add SSH Keys

SSH keys allow you to connect securely without typing passwords each time.

### Generate SSH Keys and Add to Your Account

If you don't have SSH keys yet, follow the [SSH Key Generation Guide](https://docs.runpod.io/pods/configuration/use-ssh#connect-to-a-pod-with-ssh).

1. Copy your public key (usually `~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`).
1. Go to **Settings** → **SSH Keys** in your RunPod dashboard.
1. Add your public key. This will ensure you can connect to any pod you deploy without passwords.

## Step 3: Set Up VSCode Remote SSH

1. Open VSCode

1. Install the **Remote - SSH** extension:

   - Click Extensions icon (or press `Ctrl+Shift+X` / `Cmd+Shift+X`)
   - Search for "Remote - SSH"
   - Install the extension by Microsoft

1. Configure SSH (optional but recommended):

   - Open your SSH config file: `~/.ssh/config`
   - This will make connections easier to manage

### Step 4: Deploy a Pod

1. Log in to your RunPod dashboard.

1. Click **Pods** → **Deploy**.

1. Choose your pod configuration:

   - **GPU Type**: Select based on your needs (RTX 4090, A100, etc.).
   - **Template**: Select a template that matches your project (PyTorch, TensorFlow, etc.).
   - Choose the **On-Demand** option.

1. **Environment Variables (Optional but Recommended)**:
   Click **Edit** → **Environment Variables** (at the bottom) and use the **Raw Editor**. Paste the following, replacing placeholders with your actual secrets stored in RunPod:

   ```bash
   HF_TOKEN=<your_huggingface_token>
   AWS_ACCESS_KEY_ID={{ RUNPOD_SECRET_AWS_ACCESS_KEY_ID }}
   AWS_SECRET_ACCESS_KEY={{ RUNPOD_SECRET_AWS_SECRET_ACCESS_KEY }}
   AWS_ENDPOINT_URL_S3={{ RUNPOD_SECRET_AWS_ENDPOINT_URL_S3 }}
   AWS_ENDPOINT_URL_IAM={{ RUNPOD_SECRET_AWS_ENDPOINT_URL_IAM }}
   AWS_REGION={{ RUNPOD_SECRET_AWS_REGION }}
   GITHUB_ACCESS_TOKEN={{ RUNPOD_SECRET_GITHUB_ACCESS_TOKEN }}
   WANDB_API_KEY={{ RUNPOD_SECRET_WANDB_API_KEY }}
   ```

1. Click **Deploy On-Demand** and wait 1-2 minutes for the pod to start.

## Step 5: Connect to Your Pod via SSH

### Get SSH Connection Details

1. Once your pod is running, click the **Connect** button.
1. Copy the SSH command shown (format: `ssh root@<hostname> -p <port> -i ~/.ssh/<key-name>`).

### Connect via Terminal (Quick Test)

Paste the SSH command in your terminal:

```bash
ssh root@<hostname> -p <port> -i ~/.ssh/<key-name>
```

You should connect without a password prompt if your SSH key was configured correctly.

### Connect via VSCode

**Method 1: Direct Connection**

1. In VSCode, press `F1` or `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (macOS)
1. Type "Remote-SSH: Connect to Host"
1. Paste your full SSH command: `ssh root@your-pod-id.ssh.runpod.io -p 12345`
1. Select platform: **Linux**
1. VSCode will connect and open a new window

**Method 2: Using SSH Config (Recommended)**

1. Press `F1` and select "Remote-SSH: Open SSH Configuration File"
1. Add your pod configuration:
   ```
   Host <hostname or ip>
       HostName <hostname or ip>
       User root
       Port <port>
   ```
1. Save the file
1. Press `F1` → "Remote-SSH: Connect to Host" → Select "<hostname or ip>"

## Step 6: Start Working

1. **Open a folder**: File → Open Folder → `/workspace` (this is the persistent volume).

1. **Clone the repository**:

   ```bash
   git clone https://${GITHUB_ACCESS_TOKEN}@github.com/burkimbia/repository.git
   cd repository
   ```

1. **Set up the environment**:
   You can use `uv sync` but it is recommended to install directly from system:

   ```bash
   # Install dependencies
   uv pip install -e . --python /usr/bin/python3 --break-system-packages
   ```

1. **Configure Environment Variables**:
   Create a `.env` file in the root directory:

   ```bash
   cp .env.example .env
   # Edit .env with your HF_TOKEN and WANDB_API_KEY
   ```

1. **Verify GPU Availability**:
   Run this command to ensure the GPU is detected:

   ```bash
   nvidia-smi
   ```

1. **Start coding!**

## Important Tips

### Managing Costs

- **Stop vs. Terminate**:
  - **Stop**: Releases the GPU (you stop paying for compute) but **keeps the disk volume** (you still pay a small hourly fee for storage). Your data is safe.
  - **Terminate**: Deletes the pod and the disk volume. **All billing stops, but all data is erased** unless it's on a Network Volume.
- **Use Spot instances** for cheaper rates (can be interrupted).

### Data Persistence

- Container storage is **ephemeral** - deleted when pod terminates
- Use **Network Volumes** for important data you want to keep
- Push code changes to Git regularly

### Troubleshooting

**Connection refused or timeout:**

- Verify pod is in "Running" state (not Starting or Stopped)
- Check you're using the correct port number
- Ensure your SSH key was selected when deploying

**Permission denied:**

- Verify your public key is correctly added in RunPod settings
- Check your private key permissions: `chmod 600 ~/.ssh/id_ed25519`

**Pod is expensive:**

- Choose a cheaper GPU or CPU pod for development/testing
- Remember to stop pods when taking breaks

## Useful Resources

- [VSCode Remote SSH Documentation](https://code.visualstudio.com/docs/remote/ssh)
- [VSCode SSH Tutorial](https://code.visualstudio.com/docs/remote/ssh-tutorial)
- [RunPod Documentation](https://docs.runpod.io/)

**Remember**: Always stop or terminate your pods when you're done to avoid unnecessary charges!
