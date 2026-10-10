//Start n8n in Git Bash:
docker compose up -d
//Stop n8n
docker compose down

//Export workflows
MSYS_NO_PATHCONV=1 docker compose exec n8n n8n export:workflow --backup --output=/workflows/

Make sure your terminal is in the project folder first. The prompt should end in ...\Desktop\n8n>. If it doesn't, run cd $HOME\Desktop\n8n.

It's worth adding a line to notes.md like: "Git Bash needs MSYS_NO_PATHCONV=1 in front of docker commands; PowerShell doesn't."