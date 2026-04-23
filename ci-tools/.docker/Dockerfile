FROM trivy:0.69.0 as trivy
FROM alpine:3.23.3
WORKDIR /deploy
ENV TRIVY_CACHE_DIR=/tmp
SHELL ["/bin/sh", "-ex", "-o", "pipefail", "-c"]

RUN addgroup -S deploy -g 1001 \
  && adduser -D -S -s /sbin/nologin -G deploy -u 1001 deploy \
  && install -d -g 1001 -o 1001 -m740 /deploy \
  && echo "https://mirror.yandex.ru/mirrors/alpine/v3.$(grep '^VERSION' /etc/os-release | cut -d'.' -f 2)/main" > /etc/apk/repositories \
  && echo "https://mirror.yandex.ru/mirrors/alpine/v3.$(grep '^VERSION' /etc/os-release | cut -d'.' -f 2)/community" >> /etc/apk/repositories \
  && apk add --no-cache bash \
  ca-certificates \
  git \
  coreutils \
  && rm -rf /var/cache/apk/*

COPY --from=trivy /contrib /contrib
COPY --from=trivy /usr/local/bin/trivy /usr/local/bin/trivy

ARG CURL_VERSION
ARG CURL_SHA256SUM
RUN wget -qO /usr/local/bin/curl https://github.com/moparisthebest/static-curl/releases//download/v${CURL_VERSION}/curl-amd64 \
  && echo "${CURL_SHA256SUM}  /usr/local/bin/curl" | sha256sum -c - \
  && chmod +x /usr/local/bin/curl

ARG KUSTOMIZE_VERSION
ARG KUSTOMIZE_SHA256SUM
RUN wget -qO- https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz | tar xzf - -C /usr/local/bin \
  && echo "${KUSTOMIZE_SHA256SUM}  /usr/local/bin/kustomize" | sha256sum -c -

ARG KUBECONFORM_VERSION
ARG KUBECONFORM_SHA256SUM
RUN wget -qO- https://github.com/yannh/kubeconform/releases/download/v${KUBECONFORM_VERSION}/kubeconform-linux-amd64.tar.gz | tar xzf - -C /usr/local/bin \
  && echo "${KUBECONFORM_SHA256SUM}  /usr/local/bin/kubeconform" | sha256sum -c -

ARG YQ_VERSION
ARG YQ_SHA256SUM
RUN wget -qO /usr/local/bin/yq  https://github.com/mikefarah/yq/releases/download/v${YQ_VERSION}/yq_linux_amd64 \
  && echo "${YQ_SHA256SUM}  /usr/local/bin/yq" | sha256sum -c - \
  && chmod +x /usr/local/bin/yq

ARG JQ_VERSION
ARG JQ_SHA256SUM
RUN wget -qO /usr/local/bin/jq https://github.com/stedolan/jq/releases/download/jq-${JQ_VERSION}/jq-linux64 \
  && echo "${JQ_SHA256SUM}  /usr/local/bin/jq" | sha256sum -c - \
  && chmod +x /usr/local/bin/jq

ARG TFLINT_VERSION
ARG TFLINT_SHA256SUM
ARG TFLINT_TF_RULES_VERSION
RUN wget -qO /tmp/tflint.zip https://github.com/terraform-linters/tflint/releases/download/v${TFLINT_VERSION}/tflint_linux_amd64.zip \
  && unzip /tmp/tflint.zip -d /usr/local/bin \
  && echo "${TFLINT_SHA256SUM}  /usr/local/bin/tflint" | sha256sum -c - \
  && rm /tmp/tflint.zip \
  && mkdir -p /home/deploy/.tflint.d/plugins \
  && wget -qO /tmp/tflint-ruleset-terraform.zip https://github.com/terraform-linters/tflint-ruleset-terraform/releases/download/v${TFLINT_TF_RULES_VERSION}/tflint-ruleset-terraform_linux_amd64.zip \
  && unzip /tmp/tflint-ruleset-terraform.zip  -d /home/deploy/.tflint.d/plugins \
  && rm /tmp/tflint-ruleset-terraform.zip

ARG FLUX_VERSION
ARG FLUX_SHA256SUM
RUN wget -qO- https://github.com/fluxcd/flux2/releases/download/v${FLUX_VERSION}/flux_${FLUX_VERSION}_linux_amd64.tar.gz | tar xzf - -C /usr/local/bin \
  && echo "${FLUX_SHA256SUM}  /usr/local/bin/flux" | sha256sum -c - \
  && chmod +x /usr/local/bin/flux

ARG CRANE_VERSION
ARG CRANE_SHA256SUM
RUN wget -qO- https://github.com/google/go-containerregistry/releases/download/v${CRANE_VERSION}/go-containerregistry_Linux_x86_64.tar.gz | tar xzf - -C /usr/local/bin  crane \
  && echo "${CRANE_SHA256SUM}  /usr/local/bin/crane" | sha256sum -c - \
  && chmod +x /usr/local/bin/crane

ARG COSIGN_VERSION
ARG COSIGN_SHA256SUM
RUN wget -qO /usr/local/bin/cosign https://github.com/sigstore/cosign/releases/download/v${COSIGN_VERSION}/cosign-linux-amd64 \
  && echo "${COSIGN_SHA256SUM}  /usr/local/bin/cosign" | sha256sum -c - \
  && chmod +x /usr/local/bin/cosign

ARG HELM_VERSION
ARG HELM_SHA256SUM
RUN wget -qO- https://helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz | tar xzf - -C /usr/local/bin/ --strip-components 1 "linux-amd64/helm" \
  && echo "${HELM_SHA256SUM}  /usr/local/bin/helm" | sha256sum -c - \
  && chmod +x /usr/local/bin/helm


USER 1001
