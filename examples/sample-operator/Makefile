all: fmt check

VERSION ?= v0.0.3
export VERSION := $(VERSION)

# Image registry variables
QUAY_USER ?= $(USER)
IMAGE_REGISTRY ?= quay.io/$(QUAY_USER)
IMAGE_TAG ?= latest
OPERATOR_IMAGE ?= sample-operator
HTTP_SERVER_IMAGE ?= sample-http-server
DEPLOY_DIR ?= manifests

TARGETS = \
	gen-k8s \
	goimports \
	goimports-check \
	vet \
	whitespace \
	whitespace-check

export GOFLAGS=-mod=vendor
export GO111MODULE=on

GINKGO_EXTRA_ARGS ?=
GINKGO_ARGS ?= --v -r --progress $(GINKGO_EXTRA_ARGS)
GINKGO ?= build/_output/bin/ginkgo

OPERATOR_SDK ?= build/_output/bin/operator-sdk

# Make does not offer a recursive wildcard function, so here's one:
rwildcard=$(wildcard $1$2) $(foreach d,$(wildcard $1*),$(call rwildcard,$d/,$2))

# Gather needed source files and directories to create target dependencies
directories := $(filter-out ./ ./vendor/ ,$(sort $(dir $(wildcard ./*/))))
all_sources=$(call rwildcard,$(directories),*) $(filter-out $(TARGETS), $(wildcard *))
cmd_sources=$(call rwildcard,cmd/,*.go)
pkg_sources=$(call rwildcard,pkg/,*.go)
apis_sources=$(call rwildcard,pkg/apis,*.go)

fmt: whitespace goimports

goimports: $(cmd_sources) $(pkg_sources)
	go run ./vendor/golang.org/x/tools/cmd/goimports -w ./pkg ./cmd
	touch $@

whitespace: $(all_sources)
	./hack/whitespace.sh --fix
	touch $@


check: whitespace-check vet goimports-check test-unit

whitespace-check: $(all_sources)
	./hack/whitespace.sh
	touch $@

goimports-check: $(cmd_sources) $(pkg_sources)
	go run ./vendor/golang.org/x/tools/cmd/goimports -d ./pkg ./cmd
	touch $@

$(GINKGO): go.mod
	GOBIN=$$(pwd)/build/_output/bin/ go install ./vendor/github.com/onsi/ginkgo/ginkgo

$(OPERATOR_SDK): go.mod
	GOBIN=$$(pwd)/build/_output/bin/ go install ./vendor/github.com/operator-framework/operator-sdk/cmd/operator-sdk

gen-k8s: $(OPERATOR_SDK) $(apis_sources)
	$(OPERATOR_SDK) generate k8s
	GOFLAGS=-mod= ./hack/update-codegen.sh
	touch $@

test-unit: $(GINKGO) $(KUBEBUILDER_DIR)
	$(GINKGO) $(GINKGO_ARGS) ./pkg/ ./cmd/

test-functional:
	automation/test.sh

test: test-unit test-functional

operator-build:
	docker build -f build/operator/Dockerfile -t $(IMAGE_REGISTRY)/$(OPERATOR_IMAGE):$(IMAGE_TAG) .

http-server-build:
	docker build -f build/http-server/Dockerfile -t $(IMAGE_REGISTRY)/$(HTTP_SERVER_IMAGE):$(IMAGE_TAG) .

docker-build: operator-build http-server-build

operator-push:
	docker push $(IMAGE_REGISTRY)/$(OPERATOR_IMAGE):$(IMAGE_TAG)

http-server-push:
	docker push $(IMAGE_REGISTRY)/$(HTTP_SERVER_IMAGE):$(IMAGE_TAG)

docker-push: operator-push http-server-push

vendor:
	go mod tidy
	go mod vendor

vet: $(cmd_sources) $(pkg_sources)
	go vet ./pkg/... ./cmd/...
	touch $@

gen-manifests:
	DEPLOY_DIR=$(DEPLOY_DIR) \
	CONTAINER_PREFIX=$(IMAGE_REGISTRY) \
	IMAGE_TAG=$(IMAGE_TAG) \
	OPERATOR_IMAGE=$(OPERATOR_IMAGE) \
	SERVER_IMAGE=$(HTTP_SERVER_IMAGE) \
		./hack/generate-manifests.sh

cluster-up:
	./cluster/up.sh

cluster-down:
	./cluster/down.sh

cluster-sync: cluster-operator-push cluster-operator-install

cluster-operator-push:
	./cluster/operator-push.sh

cluster-operator-install:
	./cluster/operator-install.sh

cluster-clean:
	./cluster/clean.sh

.PHONY: \
	all \
	check \
	docker-build \
	docker-push \
	test-unit \
	test-functional \
	gen-k8s \
	vendor