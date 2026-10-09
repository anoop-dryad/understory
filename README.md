# understory
Understory is a user-scoped AI agent that answers support questions about devices and issues control commands to field hardware, with every action authenticated, authorized, and audited. It extends the canopy capstone with real user management, carrying forward one invariant: the LLM never holds authority — identity lives on the transport, authorization is enforced at the backend, and structure enforces what prompts only request.


# steps

project structure using django, but dependency managemnt using uv

uv init --bare
uv add django
uv run django-admin startproject config ums
cd ums && uv run python manage.py startapp users
