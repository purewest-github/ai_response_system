FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    POETRY_VERSION=1.4.2 \
    POETRY_HOME="/opt/poetry" \
    POETRY_VIRTUALENVS_CREATE=false \
    POETRY_CACHE_DIR=/root/.cache/pypoetry

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        default-libmysqlclient-dev \
        pkg-config \
        git \
    && rm -rf /var/lib/apt/lists/*

RUN curl -sSL https://install.python-poetry.org | python3 -
ENV PATH="\${POETRY_HOME}/bin:\$PATH"

COPY pyproject.toml poetry.lock ./
RUN poetry install

COPY . .

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]