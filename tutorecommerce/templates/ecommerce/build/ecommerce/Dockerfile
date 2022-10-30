FROM docker.io/ubuntu:20.04

{% if APT_PROXY -%}
RUN echo 'Acquire::{{ APT_PROXY | url_part('scheme') }}::Proxy "{{ APT_PROXY }}";' \
    > /etc/apt/apt.conf.d/00proxy 
{%- endif %}

RUN apt update && \
  apt install -y curl git-core language-pack-en libmysqlclient-dev libssl-dev python3 python3-pip python3-venv

{% if APT_PROXY -%}
RUN rm /etc/apt/apt.conf.d/00proxy
{%- endif %}

ARG APP_USER_ID=1000
RUN useradd --home-dir /openedx --create-home --shell /bin/bash --uid ${APP_USER_ID} app
USER ${APP_USER_ID}

# Create python venv
RUN python3 -m venv /openedx/venv/
ENV PATH "/openedx/venv/bin:$PATH"

ENV PIP_INDEX_URL={{ PIP_INDEX_URL }} \
    PIP_TRUSTED_HOST={{ PIP_INDEX_URL | url_part('netloc') }}
RUN pip install setuptools==62.1.0 pip==22.0.4 wheel==0.37.1

# Install a recent version of nodejs
RUN pip install nodeenv
RUN nodeenv /openedx/nodeenv --node=16.14.2 --prebuilt
ENV PATH /openedx/nodeenv/bin:${PATH}

# Install ecommerce
ARG ECOMMERCE_REPOSITORY=https://github.com/edx/ecommerce.git
ARG ECOMMERCE_VERSION={{ OPENEDX_COMMON_VERSION }}
RUN mkdir -p /openedx/ecommerce && \
    git clone $ECOMMERCE_REPOSITORY --branch $ECOMMERCE_VERSION --depth 1 /openedx/ecommerce
WORKDIR /openedx/ecommerce

# Identify tutor user to cherry-pick commits
RUN git config --global user.email "tutor@overhang.io" \
  && git config --global user.name "Tutor"

# nodejs requirements (aka: "make requirements.js")
ARG NPM_REGISTRY={{ NPM_REGISTRY }}
RUN npm install --verbose --registry=$NPM_REGISTRY
RUN ./node_modules/.bin/bower install --allow-root

# python requirements
RUN pip install -r requirements.txt
RUN pip install uwsgi==2.0.20

# Install private requirements: this is useful for installing custom payment processors.
COPY --chown=app:app ./requirements/ /openedx/requirements
RUN cd /openedx/requirements/ \
  && touch ./private.txt \
  && pip install -r ./private.txt

{% for extra_requirement in ECOMMERCE_EXTRA_PIP_REQUIREMENTS %}RUN pip install {{ extra_requirement }}
{% endfor %}

# Collect static assets (aka: "make static")
COPY --chown=app:app assets.py ./ecommerce/settings/assets.py
ENV DJANGO_SETTINGS_MODULE ecommerce.settings.assets
RUN python3 manage.py update_assets --skip-collect
RUN ./node_modules/.bin/r.js -o build.js
RUN python3 manage.py collectstatic --noinput
RUN python3 manage.py compress --force

# unset unneeded environment variables
ENV PIP_INDEX_URL= \
    PIP_TRUSTED_HOST= 

# Setup minimal yml config file, which is required by production settings
RUN echo "{}" > /openedx/config.yml
ENV ECOMMERCE_CFG /openedx/config.yml

EXPOSE 8000
CMD uwsgi \
    --static-map /static=/openedx/ecommerce/assets \
    --static-map /media=/openedx/ecommerce/course_ecommerce/media \
    --http 0.0.0.0:8000 \
    --thunder-lock \
    --single-interpreter \
    --enable-threads \
    --processes=2 \
    --buffer-size=8192 \
    --wsgi-file ecommerce/wsgi.py
