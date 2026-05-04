Viswa Linux Learning Journey

This repository is a structured collection of learning materials and documentation focused on Linux Kernel Internals, Drivers, and Architecture.

📂 Repository Structure
The project is organized into specialized "Books" covering key kernel subsystems:

Core Architecture: Kernel_Architecture_Boot_Book, Process_Scheduling_Book, Memory_Management_Book.
Driver Development: Device_Drivers_Book, Device_Framework_Book, Interrupt_Concurrency_Book.
Systems & Infrastructure: File_Systems_Book, Networking_Book, Power_Management_Book, Security_Book.
Advanced Topics: RTOS_Book, Containers_Virtualization_Book, Build_Debug_Book.


1. Setup SSH-RSA Key
If you specifically need RSA (though your history shows you successfully used Ed25519),
use these commands
#bash Generate a new 4096-bit RSA key
ssh-keygen -t rsa -b 4096 -C "my_mailID@gmail.com" -f ~/.ssh/id_rsa_viswa

# Start the ssh-agent and add your new key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa_viswa

# Copy the public key to your clipboard to add to GitHub Settings
cat ~/.ssh/id_rsa_viswa.pub

2. Clone the Repository
Since you are using a custom key, ensure your ~/.ssh/config has an entry for github.com pointing to that specific file.
Then clone using the SSH URL:
#bash Clone using SSH
git clone git@github.com:viswa-techie/viswa_linux.git
cd viswa_linux

3. Update README, Commit, and Push
Use these commands to apply the README content we discussed and sync it with GitHub:

# 1. Create or update the README file
# (You can also use 'nano README.md' to paste the full content from the previous response)
echo "# Viswa Linux Learning Journey" > README.md

# 2. Stage the file
git add README.md

# 3. Commit with a message
git commit -m "Update README with project structure"

# 4. Push to GitHub
git push origin master
