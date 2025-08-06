# Update package list
sudo apt update

# Install prerequisites
sudo apt install -y wget apt-transport-https

# Add Microsoft package repository
wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# Update package list again
sudo apt update

# Install .NET runtime (or SDK if you need to build on the VM)
sudo apt install -y dotnet-runtime-8.0