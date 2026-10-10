//Start n8n in Git Bash:
docker compose up -d
//Stop n8n
docker compose down

//Export workflows
MSYS_NO_PATHCONV=1 docker compose exec n8n n8n export:workflow --backup --output=/workflows/
