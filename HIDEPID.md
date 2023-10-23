# hidepid

The `hidepid` is an option that can be set on the proc filesystem (`/proc`) in Linux to restrict users from seeing each other’s processes. Here’s a brief overview and how you can implement it:

## hidepid Options:
- `hidepid=0`: This is the default setting where all users can see all processes.
- `hidepid=1`: Users can see their own processes and certain details about other users’ processes (like process owners).
- `hidepid=2`: Users can only see their own processes.

## Implementing hidepid:

1. **Edit `/etc/fstab` file to include the hidepid option:**
    ```bash
    sudo nano /etc/fstab
    ```

2. **Add or modify the following line:**
    ```plaintext
    proc /proc proc defaults,hidepid=2 0 0
    ```
   
3. **Remount the proc filesystem:**
    ```bash
    sudo mount -o remount /proc
    ```
   
4. **Verify the Changes:**
    ```bash
    ps aux
    ```
   You should only be able to see the processes of the logged-in user.

5. **Reboot to make sure the changes persist:**
    ```bash
    sudo reboot
    ```
