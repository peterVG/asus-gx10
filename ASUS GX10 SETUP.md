# ASUS GX10 SETUP

## Gemma 4 31B model
To test the performance and accuracy of ***OCR***, ***text classification***, and ***image classification***, the absolute best model for the Asus GX10 (Blackwell) setup is Gemma 4 31B (Dense).

While you could use the smaller E4B or 26B MoE variants, the 31B Dense model is the flagship "frontier" version released in April 2026. It is specifically architected for high-precision document and visual tasks.

For the Asus GX10, don't compromise. Use the 31B Dense model at FP16 for the highest possible accuracy in OCR and classification. The Blackwell chip was built precisely for this level of unquantized multimodal performance.

### Gemma 4 31B for OCR

***Dense vs. MoE:*** While the 26B MoE is faster, the 31B Dense is the highest-quality open model in the Gemma family. For records management and legal-grade archival tasks, you want the raw reasoning power of every parameter.

***Visual Fidelity:*** Running in FP16 ensures that OCR tasks don't suffer from the "bit-smearing" found in 4-bit/8-bit quantizations. When you set the visual token budget to 1120, the model sees document details at a "retinal" level. Unlike many VLMs that have a fixed number of image tokens, Gemma 4 allows you to set a budget from 70 to 1120 tokens. For high-accuracy OCR, you can set the budget to 1120, giving the model enough "retinal detail" to read small text and complex document layouts.

***OmniDocBench Scores:*** In recent 2026 benchmarks, the 31B Dense model achieved an edit distance of 0.131 (lower is better), significantly beating the 26B MoE (0.149) and the E4B (0.181).

***Thinking Mode:*** You can now keep the <|think|> logic enabled globally without worrying about latency. By enabling the <|think|> token, you can force the model to "reason" through the document layout (e.g., "This is a three-column invoice; I should read from top-left first") before it transcribes, which drastically reduces hallucinations in structured data extraction. The Blackwell chip processes the model's internal "chain-of-thought" fast enough that it feels like standard generation.

### Gemma 4 31B for Image & Text Classification
For classification, you need a model that understands the relationship between visual features and high-level semantic categories.

***MMMU Pro Lead:*** The 31B Dense leads the Gemma 4 family with a 76.9% score on MMMU Pro (multimodal understanding), making it the most accurate choice for classifying complex images (e.g., distinguishing between different types of historical records or mechanical parts).

***Text Classification Mastery:*** Because the vision encoder is "fused" with a 31B parameter dense LLM, it outperforms smaller models at categorizing text based on subtle context. It can easily classify a document as "Sensitive/Legal" vs. "Public/Administrative" by understanding the tone of the text, not just keywords.

### Gemma 4 31B for MMLU Pro
The MMLU Pro benchmark is a comprehensive evaluation of multimodal reasoning, covering 57 tasks across STEM, humanities, and social sciences. It tests the model's ability to understand and reason about complex visual and textual information. 

***Top Performance:*** The 31B Dense model leads the Gemma 4 family with a 76.9% score on MMMU Pro, significantly outperforming the 26B MoE (74.6%) and the E4B (71.0%). This makes it the best choice for multimodal reasoning tasks.

## System & CUDA Optimization
Unlike macOS, the Ubuntu-based DGX distro requires explicit driver-to-container orchestration. The GX10 uses NVLink-C2C, which makes CPU and GPU memory act as one giant pool.

### Verify Blackwell Status
Run the NVIDIA System Management Interface to ensure the GB10 is recognized and idling at the correct wattage.
> nvidia-smi

Note: The Blackwell GPU should show 128GB of available VRAM.

### Configure NVIDIA Container Toolkit
Since you’ll likely run Open WebUI via Docker, ensure the GPU is "passed through" correctly.

### Verify the runtime is installed
> nvidia-ctk --version

### If missing, install to allow Docker to see Blackwell
> sudo apt-get install -y nvidia-container-toolkit
> sudo systemctl restart docker

## Maximum Gemma Strategy (Blackwell Upgrade)
On the Asus GX10, you can move beyond the "Small Assistant" constraints required for mobile silicon. With the NVIDIA Blackwell Superchip and 128GB of Unified Memory, you have the headroom to run the flagship Gemma 4 31B Dense model at full precision (FP16) without needing a secondary task model.

### Recommended "Power" Setup
* Main Model: gemma4:31b-fp16
* Memory Footprint: ~62GB (Leaves 66GB for context and system overhead)
* Context Window: 256k tokens (Blackwell’s architecture handles massive long-form archival documents natively).
* Performance Expectation: 40–60 tokens/sec (With Blackwell’s NVFP4 precision mode via NVIDIA Model Optimizer, you can push this even higher while maintaining FP16-equivalent accuracy.)

## Deployment Command (on Asus)
> ollama run gemma4:31b-fp16

## Open WebUI (DGX Architecture)
Running Open WebUI on Ubuntu DGX is best done via Docker Compose to ensure the ollama and open-webui containers share the NVLink-C2C memory space.

***Deploying the Stack***

Create a docker-compose.yaml:
YAML

services:
  ollama:
    image: ollama/ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434

***Launch:***
> docker compose up -d

## Testing Configuration for OCR (via API or Open WebUI):
To get the most accurate OCR results, use these parameters in your request:
* Image Token Budget: 1120 (Maximum detail)
* Temperature: 0.0 (For deterministic, literal transcription)
* System Prompt: "<|think|> You are a high-precision OCR agent. Transcribe every word exactly as it appears, preserving the layout."

Note: With 128GB VRAM, you can set the Context Window to 256,000 tokens without breaking a sweat.

## Testing with Hugging Face Datasets
***Install the Hugging Face CLI***
> curl -LsSf https://hf.co/cli/install.sh | bash
> source ~/.bashrc

***Download a Testing Dataset***
For archival testing (records management), I recommend the Enron Email Dataset or a CommonCrawl subset.
Download a specific dataset via terminal:
> hf download datasets/allenai/c4 --repo-type dataset --limit 10GB

***Explanation:** hf download is the fastest way to pull files. The --limit flag ensures you don't accidentally pull the entire multi-terabyte C4 dataset.

***Local Ingestion***
Once downloaded, these files are usually in .parquet or .jsonl format. You can point your Open WebUI "Documents" feature to the local path: ~/.cache/huggingface/hub/datasets--[name]/snapshots/

## Using Custom Data
To keep your custom data organized and separate from model weights or Hugging Face caches, create a dedicated data volume in your home directory.

***Create the Directory Structure:**
> mkdir -p ~/datasets/custom/ocr_tests
> mkdir -p ~/datasets/custom/classification_samples

***Why this location?***
- Security: Storing data within your user home directory (~/) ensures it inherits your user permissions. 
- Speed: Keeping datasets on the local SSD (rather than an external or network drive) is critical for high-speed indexing and vectorization. 
- Portability: This structure makes it easy to point Open WebUI or custom Python scripts to a single source of truth. 

## Loading Data into AI Model
Once the files are on the Asus, you have two primary ways to "feed" them to the Gemma 4 31B model.

***Method 1: Via Open WebUI "Documents" (RAG)***
For testing how well the model retrieves information from large archival sets:
1. Open Open WebUI in your browser (http://[ASUS_IP_ADDRESS]:3000). 
2. Navigate to Workspace > Documents. 
3. Instead of uploading via the browser (which is slower), use the local path feature if your Docker container is mapped to your home directory. 
If not, use the Upload button to select the files you just moved to the Asus.

***Method 2: Direct Injection for OCR Testing***
For the high-precision OCR tests you are planning, you should point the model directly to the raw image files to avoid any compression from web uploads.
* When using the chat interface, use the upload icon to attach a specific sample from the ~/datasets/custom/ocr_tests folder.
* Ensure your Visual Token Budget is set to 1120 in the model settings to maintain "retinal" detail during the load. 

## Dataset Maintenance
Custom datasets can quickly bloat. Use these commands on the Asus to monitor and prune them.

***Check Dataset Size:***
> du -sh ~/datasets/custom/* [cite: 203]

***Remove Old Tests:***
> rm -rf ~/datasets/custom/old_test_directory [cite: 182]

Store your ground-truth labels (the "correct" OCR text) in a .jsonl file alongside your images in the same directory. This makes it easier to automate accuracy scoring later.

## 1TB+ DATASETS
Testing datasets larger than 1TB on the Asus GX10 requires leveraging the high-speed USB-C/Thunderbolt ports to bridge external storage with the Blackwell-powered AI stack. To maintain the performance expected of a DGX-class workstation, you must properly mount the drive and implement a staging strategy to prevent the USB interface from becoming a bottleneck during Gemma 4 31B inference.

### Mounting the External SSD (Ubuntu DGX)
Unlike a Mac, the Asus GX10 (Ubuntu) requires manual mounting to ensure the storage remains persistent and accessible to Docker/Ollama.

***Identify the Drive:***
Plug in your SSD and find its device identifier:
> lsblk
Look for a disk (likely sdb or nvme1n1) matching your SSD's size.

***Create a Mount Point:***
> sudo mkdir -p /mnt/external_data

***Mount the Drive:***
Replace 'sdb1' with your actual partition name
> sudo mount /dev/sdb1 /mnt/external_data

***Set Permissions:***
Ensure your user and the Docker service can read the files:
> sudo chown -R $USER:$USER /mnt/external_data
> sudo chmod -R 755 /mnt/external_data

## External USB SSD - The "Staging Area" Cache Strategy
To avoid the read/write delays of USB during active testing, use a Staging Area on your internal 1TB NVMe. This allows you to pull a "batch" of files (e.g., 100GB of images for OCR) into the fast local cache, process them, and then cycle in the next batch.

***Setup the Local Cache:***
> mkdir -p ~/datasets/staging_cache

***Automated Pre-loading (Bash Script):***
Create a script to "cycle" data from the external SSD to the internal cache:
> nano ~/cycle_data.sh

Paste this logic:
```
#!/bin/bash
# Usage: ./cycle_data.sh [subdirectory_name]

SOURCE="/mnt/external_data/$1"
DEST="$HOME/datasets/staging_cache"

echo "Clearing old cache..."
rm -rf "$DEST"/*

echo "Pre-loading new batch from SSD to local NVMe..."
# rsync is used here to ensure data integrity during the transfer
rsync -av --progress "$SOURCE" "$DEST"

echo "Staging complete. Blackwell can now ingest at internal NVMe speeds."
```

***Explanation:***
This ensures that while Gemma 4 is performing high-precision OCR, it is pulling from the local 1TB NVMe, not the slower USB bus.

## Connecting the Cache to Open WebUI
To make these files visible to your AI models, you must "map" the staging directory into your Open WebUI Docker container.

***Modify your docker-compose.yaml:***
services:
   open-webui:
   # ... other config ...
      volumes:
         - open-webui:/app/data
         - /home/peter/datasets/staging_cache:/app/data/STAGING_DATA:ro

Note: The :ro flag ensures the AI cannot accidentally delete your source data.

***Restart the Stack:***
> docker compose up -d

***Summary of STORAGE CACHE Workflow:***
1. Mount the 2TB+ SSD to /mnt/external_data.
2. Stage a specific test batch using ./cycle_data.sh.
3. Run high-precision OCR or classification on the STAGING_DATA folder in Open WebUI. 
4. Analyze performance, then cycle the next batch from the SSD. 
This approach allows you to theoretically test datasets of any size (4TB, 8TB+) while keeping the Gemma 4 31B model running at its peak 40–60 tokens/sec performance.

## Monitoring The DGX Way
While ollama ps still works, you have professional-grade tools available on the GX10.

### NVTop (Visual GPU Monitor)
Install nvtop for an "Activity Monitor" style view of your Blackwell GPU.
> sudo apt install nvtop
> nvtop

Watch the Memory Bar: With 128GB, you should see plenty of green space even with 70B models running.

### NVIDIA DCGM (Diagnostic Check)
If the GX10 feels slow, check for "throttling" or "XID errors":

> nvidia-smi -q -d PERFORMANCE

## Monitoring the Pipeline
When running tests on massive datasets, monitor the data flow to ensure the SSD isn't lagging:

***Monitor Disk I/O:***
> iostat -xz 1

***Monitor VRAM Ingestion:***
Use
> nvitop
to see how fast the Blackwell chip is swallowing the staged data. 

## Remote Login

***Phase 1: Key Generation (On your MacBook Air M5)***
First, you need to create your unique "digital fingerprint" on your Mac.
1. Open Terminal on your MacBook.
2. Generate a modern Ed25519 key:
> ssh-keygen -t ed25519 -C "peter@macbook-air-m5"
    
* Note: When asked where to save, just press Enter (default).
* Note: When asked for a passphrase, you can leave it empty for "auto-login" or provide one for extra security. In your local setup, many prefer empty for a seamless experience.

3. Verify the keys exist:
> ls -l ~/.ssh/id_ed25519*(You should see a private file and a .pub public file).

***Phase 2: Initial Asus Configuration (On the Asus GX10)***
You need to do a one-time "open door" setup so the Asus can receive your key.
1. Install OpenSSH Server:
> sudo apt update && sudo apt install -y openssh-server
2. Start and Enable the Service:
> sudo systemctl enable --now ssh
3. Find your Asus IP Address:
> ip addr show | grep "inet " | grep -v "127.0.0.1"Look for the number like 192.168.x.xx.

***Phase 3: The "Handshake" (From your MacBook Air)***
This is the only time you will ever have to type your Asus password.
1. Transfer the Public Key to the Asus:
> ssh-copy-id -i ~/.ssh/id_ed25519.pub peter@[ASUS_IP_ADDRESS]
    
* Replace peter with your Asus username.
* Replace [ASUS_IP_ADDRESS] with the number from Phase 2.
* Type your Asus password when prompted.

***Phase 4: Locking the Door (On the Asus GX10)***
Now that your Mac has the "key," we can tell the Asus to never ask for a password again and to ignore anyone who doesn't have a key.
1. Edit the SSH Configuration:
> sudo nano /etc/ssh/sshd_config

2. Change/Add these two lines (Use arrow keys to find them, remove the # if present):
    * PubkeyAuthentication yes
    * PasswordAuthentication no
3. Save and Exit: Press Ctrl + O, then Enter, then Ctrl + X.
4. Restart SSH to apply changes:
> sudo systemctl restart ssh

***Phase 5: The "One-Click" Alias (On your MacBook)***
Typing ssh peter@192.168... every time is tedious. Let's make it a single word.
1. Create an SSH Config File:Bash
> nano ~/.ssh/config
2. Paste this configuration:
Host asus
    HostName [ASUS_IP_ADDRESS]
    User peter
    IdentityFile ~/.ssh/id_ed25519Save and Exit: Ctrl + O, Enter, Ctrl + X.

***The Result***
From now on, whenever you want to control your Blackwell setup from your Mac, you simply type:

ssh asus

You will be logged in instantly without a password prompt.

### Continuous Remote Monitoring
While logged in via ssh asus, keep a separate terminal tab open running this to monitor the Blackwell GPU in real-time:

> watch -n 1 nvidia-smi

Or, for the high-fidelity view:

> nvitop

## ASUS GX10 Maintenance
***Model Storage: Where the Weights Live***
Ollama and Hugging Face store data in different hidden directories on the Ubuntu DGX filesystem.

Ollama | /usr/share/ollama/.ollama/models | Active model weights and manifests.
Hugging Face |~/.cache/huggingface/hub | Downloaded datasets and raw model files.
Open WebUI | /open-webui/data | Vector database (ChromaDB) for RAG documents.

***Pruning & Cleaning Storage***
Use these commands to recover disk space on your Asus GX10.

Pruning Ollama Models
To see what’s taking up space:
> ollama list
To remove a model completely:
> ollama rm gemma4:31b-fp16

***Managing Hugging Face Cache***
Datasets can easily exceed 100GB. Use the HF CLI to scan and delete unused caches:

Scan to see what's taking up space
> huggingface-cli scan-cache

Interactively delete unused models/datasets
> huggingface-cli delete-cache

## Resource Control: Managing GPU & RAM
Sometimes a model stays "stuck" in the Blackwell VRAM even after you've closed the browser.

***Identifying "Zombie" Processes***
If nvidia-smi shows memory is still used but no chat is active, find the process ID (PID):

Find any process using the GPU:
> sudo fuser -v /dev/nvidia*

***The "Nuclear" Reset (Flush VRAM)***
If the system feels sluggish or a model won't unload, run this to force-clear the Blackwell GPU memory and the Linux system cache:

Kill any remaining Ollama runner processes:
> pkill -f "ollama runner"

Clear Linux system memory cache (releases RAM)
> sudo sh -c "sync; echo 3 > /proc/sys/vm/drop_caches"

***Monitoring Disk Usage***
When testing large document sets, keep an eye on the specific directory where your test data sits:

Check size of your test data folder:
> du -sh ~/arkrim-test-data

### The "Clean Start" Script
Create a small bash script on your Asus called clean_ai.sh to quickly reset your environment from your Mac:

> nano ~/clean_ai.sh

Paste this content:
```
#!/bin/bash
echo "Resetting Blackwell VRAM and System Cache..."
pkill -f "ollama runner"
docker restart open-webui
sudo sh -c "sync; echo 3 > /proc/sys/vm/drop_caches"
echo "System Cleaned. Ready for new Gemma 4 tests."
```

Make it executable: 
> chmod +x ~/clean_ai.sh

From your Mac: Just run 
> ssh asus "./clean_ai.sh" 
to refresh your entire workstation in seconds.
