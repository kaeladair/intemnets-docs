# Drone Setup

```bash
curl https://debian.parrot.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/debian.parrot.com.gpg
echo "deb [signed-by=/usr/share/keyrings/debian.parrot.com.gpg] https://debian.parrot.com/ $(lsb_release -cs) main generic" | sudo tee /etc/apt/sources.list.d/debian.parrot.com.list > /dev/null
sudo apt update
```
