FROM python:3.8-slim

WORKDIR /app

# Copier tous les fichiers du répertoire local vers le répertoire de travail
COPY . /app/

RUN apt-get update \
    && apt-get install gcc -y \
    && apt-get clean

RUN pip install -r /app/requirements.txt \
    && rm -rf /root/.cache/pip


# Définir le chemin Python
ENV PYTHONPATH=/app

# Exposer le port d'écoute de l'application
EXPOSE 8000

# Commande pour lancer l'application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]