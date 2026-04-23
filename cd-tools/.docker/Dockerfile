FROM alpine:3.23.3

ENV HELM_DIFF_COLOR=true \
  HELM_PLUGINS="/opt/helm/plugins" \
  HELM_MAX_HISTORY="3" \
  KUBECONFIG="/tmp/kube.conf"

# Metadata
LABEL org.label-schema.name="helm-kubectl" \
  org.label-schema.description="cd-tools helm & kubectl" \
  maintainer="infra"

SHELL ["/bin/sh", "-ex", "-o", "pipefail", "-c"]

RUN echo "https://mirror.yandex.ru/mirrors/alpine/v3.$(grep '^VERSION' /etc/os-release | cut -d'.' -f 2)/main" > /etc/apk/repositories \
  && echo "https://mirror.yandex.ru/mirrors/alpine/v3.$(grep '^VERSION' /etc/os-release | cut -d'.' -f 2)/community" >> /etc/apk/repositories \
  && apk add --no-cache git \
  && rm -rf /var/cache/apk/*

RUN mkdir -p /opt/helm/plugins/helm-secrets /opt/helm/plugins/helm-diff \
  && addgroup -S deploy -g 65532 \
  && adduser -D -S -s /sbin/nologin -G deploy -u 65532 deploy \
  && install -d -g 65532 -o 65532 -m740 /deploy \
  && install -o 65532 -m600 /dev/null ${KUBECONFIG}

ARG JQ_VERSION
ARG JQ_SHA256SUM
RUN wget -qO /usr/local/bin/jq https://github.com/stedolan/jq/releases/download/jq-${JQ_VERSION}/jq-linux64 \
  && echo "${JQ_SHA256SUM}  /usr/local/bin/jq" | sha256sum -c - \
  && chmod +x /usr/local/bin/jq

ARG CURL_VERSION
ARG CURL_SHA256SUM
RUN wget -qO /usr/local/bin/curl https://github.com/moparisthebest/static-curl/releases/download/v${CURL_VERSION}/curl-amd64 \
  && echo "${CURL_SHA256SUM}  /usr/local/bin/curl" | sha256sum -c - \
  && chmod +x /usr/local/bin/curl

ARG SOPS_VERSION
ARG SOPS_SHA256SUM
RUN wget -qO /usr/local/bin/sops https://github.com/mozilla/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64 \
  && echo "${SOPS_SHA256SUM}  /usr/local/bin/sops" | sha256sum -c - \
  && chmod +x /usr/local/bin/sops

ARG KUBECTL_VERSION
ARG KUBECTL_SHA256SUM
RUN wget -qO /usr/local/bin/kubectl  https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl \
  && echo "${KUBECTL_SHA256SUM}  /usr/local/bin/kubectl" | sha256sum -c - \
  && chmod +x /usr/local/bin/kubectl

ARG HELM_VERSION
ARG HELM_SHA256SUM
RUN wget -qO- https://helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz | tar xzf - -C /usr/local/bin/ --strip-components 1 "linux-amd64/helm" \
  && echo "${HELM_SHA256SUM}  /usr/local/bin/helm" | sha256sum -c - \
  && chmod +x /usr/local/bin/helm

ARG KUBEDOG_VERSION
ARG KUBEDOG_SHA256SUM
RUN wget -qO /usr/local/bin/kubedog https://tuf.kubedog.werf.io/targets/releases/${KUBEDOG_VERSION}/linux-amd64/bin/kubedog \
  && echo "${KUBEDOG_SHA256SUM}  /usr/local/bin/kubedog" | sha256sum -c - \
  && chmod +x /usr/local/bin/kubedog

ARG HELM_PLUGIN_DIFF_VERSION
ARG HELM_PLUGIN_DIFF_SHA256SUM
RUN wget -qO-  https://github.com/databus23/helm-diff/releases/download/v${HELM_PLUGIN_DIFF_VERSION}/helm-diff-linux-amd64.tgz | tar xzf - -C /opt/helm/plugins/helm-diff --strip-components 1 diff/bin/diff  diff/plugin.yaml \
  && echo "${HELM_PLUGIN_DIFF_SHA256SUM}  /opt/helm/plugins/helm-diff/bin/diff" | sha256sum -c -

ARG HELM_PLUGIN_SECRET_VERSION
ARG HELM_PLUGIN_SECRET_SHA256SUM

RUN mkdir -p /opt/helm/plugins/secrets && wget -qO-  https://github.com/jkroepke/helm-secrets/releases/download/v${HELM_PLUGIN_SECRET_VERSION}/secrets-${HELM_PLUGIN_SECRET_VERSION}.tgz | tar xzf - -C /opt/helm/plugins/secrets --strip-components 1 secrets/scripts/ secrets/plugin.yaml


ARG YQ_VERSION
ARG YQ_SHA256SUM
RUN wget -qO /usr/local/bin/yq  https://github.com/mikefarah/yq/releases/download/v${YQ_VERSION}/yq_linux_amd64 \
  && echo "${YQ_SHA256SUM}  /usr/local/bin/yq" | sha256sum -c - \
  && chmod +x /usr/local/bin/yq

ARG FLUX_VERSION
ARG FLUX_SHA256SUM
RUN wget -qO- https://github.com/fluxcd/flux2/releases/download/v${FLUX_VERSION}/flux_${FLUX_VERSION}_linux_amd64.tar.gz | tar xzf - -C /usr/local/bin \
  && echo "${FLUX_SHA256SUM}  /usr/local/bin/flux" | sha256sum -c - \
  && chmod +x /usr/local/bin/flux

WORKDIR /deploy

USER 1001
