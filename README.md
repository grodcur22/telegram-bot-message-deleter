# Telegram Chat History Cleaner

A lightweight Bash utility designed to automate the deletion of messages in specific Telegram chats using a Telegram Bot. This script is particularly useful for maintaining clean bot-operated channels or groups by removing recent activity logs.

---

## Features

* **Dual-Mode Deletion**:
    * **History-Based**: Scans the Telegram API `getUpdates` log to identify and delete specific recent message IDs.
    * **Brute Force/Preventive**: If no history is found, it sends a trigger message and performs a downward sweep of the last 100 message IDs to ensure the chat is cleared.
* **Multi-Chat Support**: Ability to target multiple Chat IDs in a single execution.
* **Silent Execution**: Deletion requests are processed in the background to avoid cluttering the terminal output.

---

## Prerequisites

* **Telegram Bot Token**: You need a token from [@BotFather](https://t.me/botfather).
* **Chat IDs**: The numeric IDs of the chats you wish to clean.
* **Dependencies**: Ensure `curl` and `grep` (with PCRE support, usually standard on Linux) are installed on your system.

---

## Configuration

Open the script and modify the configuration section with your credentials:

```bash
#!/bin/bash

# --- CONFIGURATION ---
# Replace these with your actual Bot Token and Chat IDs before running
TOKEN="YOUR_BOT_TOKEN_HERE"
CHATS=("CHAT_ID_1" "CHAT_ID_2")

echo "Searching for recent bot updates..."

# Get the latest available message_ids from getUpdates
# Note: getUpdates only shows messages from the last 24 hours approximately
UPDATES=$(curl -s "https://api.telegram.org/bot$TOKEN/getUpdates")

for CHAT_ID in "${CHATS[@]}"; do
    echo "-----------------------------------------------"
    echo "Cleaning chat ID: $CHAT_ID"

    # Try to extract real IDs from getUpdates using grep and regex
    MSG_IDS=$(echo "$UPDATES" | grep -oP '"message_id":\s*\K\d+' | sort -rn | uniq)

    if [ -z "$MSG_IDS" ]; then
        echo "Warning: No IDs found in getUpdates. Attempting preventive cleanup..."
        
        # Send a new message to get the current ID and start the sweep from there
        LAST_MSG=$(curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
            -d "chat_id=$CHAT_ID" \
            -d "text=Starting cleanup process...")
        
        CURRENT_ID=$(echo "$LAST_MSG" | grep -oP '"message_id":\s*\K\d+')
        
        if [ ! -z "$CURRENT_ID" ]; then
            echo "Applying brute force from ID: $CURRENT_ID"
            for (( i=0; i<100; i++ )); do
                TARGET_ID=$((CURRENT_ID - i))
                # Silent deletion of previous messages
                curl -s -X POST "https://api.telegram.org/bot$TOKEN/deleteMessage" \
                     -d "chat_id=$CHAT_ID" \
                     -d "message_id=$TARGET_ID" > /dev/null
            done
            echo "Mass cleanup attempt completed for $CHAT_ID."
        fi
    else
        # Deletion based on IDs found in recent history
        for MSG_ID in $MSG_IDS; do
            echo "Deleting message ID: $MSG_ID"
            curl -s -X POST "https://api.telegram.org/bot$TOKEN/deleteMessage" \
                 -d "chat_id=$CHAT_ID" \
                 -d "message_id=$MSG_ID" > /dev/null
        done
        echo "Cleanup by history completed."
    fi
done

echo "-----------------------------------------------"
echo "Process finished."
````

---

## How It Works

1. **Fetch Updates**: The script calls the `getUpdates` method to retrieve messages received by the bot in the last 24 hours.

2. **ID Extraction**: It parses the JSON response using Regex to find all available `message_id` values.

3. **Targeted Deletion**: If IDs are found, the script iterates through them and sends `deleteMessage` requests for each Chat ID configured.

4. **Fallback Mechanism**: If the update log is empty (common if the bot has been inactive), the script:

   * Sends a "Starting cleanup process..." message.
   * Captures the ID of that new message.
   * Calculates the 100 preceding IDs and attempts to delete them one by one.

---

## Important Notes

* **Permissions**: The bot must have "Delete Messages" permissions in the target group or channel.
* **Limitations**: Telegram's API generally allows bots to delete messages sent by others only if they are administrators. Bots can always delete their own messages.
* **API Window**: The `getUpdates` method only tracks messages from the last 24 hours or until they are acknowledged.

---

## Usage

Give execution permissions to the file:

```bash
chmod +x cleanup_script.sh
```

Run the script:

```bash
./cleanup_script.sh
```

---

## Contributing

Feel free to fork this repository and submit pull requests for any enhancements, such as adding JSON parsing via `jq` for better stability.

```
```
