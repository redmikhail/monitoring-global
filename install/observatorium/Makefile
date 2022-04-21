SHELL=bash
export OBSERVATORIUM_APPS_URL := $(shell oc get --namespace openshift-ingress-operator ingresscontrollers/default --output jsonpath='{.status.domain}{"\n"}')
export DEX_NAMESPACE?=dex
export MINIO_NAMESPACE?=observatorium-minio
export OBSERVATORIUM_NAMESPACE?=observatorium


ifndef observatorium_env
	export observatorium_env :=dev
endif

ifeq (,$(and $(shell which vault),$(shell vault print token)))
export DEX_TENANT_CLIENT_SECRET?="ZXhhbXBsZS1hcHAtc2VjcmV0"
export DEX_ADMIN_PASS?="password"
else ifeq ($(observatorium_env),stage)
export DEX_TENANT_CLIENT_SECRET?="stage"
else ifeq ($(observatorium_env),prod)
export DEX_TENANT_CLIENT_SECRET:=$(shell vault kv get -field=dex_tenant_client_secret -address=https://vault.xxx.net blah/blah)
export DEX_ADMIN_PASS?=$(shell vault kv get -field=dex_password -address=https://vault.xxx.net blah/blah-key)
else ifeq ($(observatorium_env),dev)
export DEX_TENANT_CLIENT_SECRET?="blah"
endif


# Deploy dex IDP based on environment type
.PHONY: create/dex
create/dex:
ifeq ($(observatorium_env),dev)
create/dex: create/dex/dev
endif


#Deploy Dex for Dev environment uzing kustomize. 
.PHONY: create/dex/dev
create/dex/dev:
	@echo "Create dex in: $(DEX_NAMESPACE) for Dev environment"
#	Nothing to replace but keeping the command for consistency, add token replacement based on prod 
	@cat config/dex/dev/config.yaml | \
	sed -e "s/<dex_tenant_client_secret>/$(DEX_TENANT_CLIENT_SECRET)/g"|sed 's/<dex_admin_pass_hash>/\"$(shell htpasswd -bnBC 10 "admin"  $(DEX_ADMIN_PASS) | cut -d ":" -f 2 | sed "s/2y/2a/;s/\\$$/\\\\$$/g;s/\./\\\\./g;s#\/#\\\/#g")\"/g'\
	 > config/dex/dev/config.generated.yaml
	@oc kustomize config/dex/dev |sed 's/<namespace>/$(DEX_NAMESPACE)/g' |oc apply -f -



# Deploy MINIO to different environments

.PHONY: create/minio
create/minio:
ifeq ($(observatorium_env),dev)
create/minio: create/minio/dev
endif
# ifeq ($(observatorium_env),stage)
# create/minio: create/minio/stage
# endif

.PHONY: create/minio/dev
create/minio/dev:
	@echo "create minio in: $(MINIO_NAMESPACE) for Dev environment"
	@oc kustomize config/minio/dev |sed 's/<namespace>/$(MINIO_NAMESPACE)/g' |oc apply -f -


.PHONY: create/crds
create/crds:
	@oc apply -f ./crd.yaml
	@echo "Waiting for Observatorium CRD to be created"
	@for i in {1..12}; do oc get crd observatoria.core.observatorium.io && break || sleep 5; done

.PHONY: create/operator
create/operator:
	@echo "Create observatorium operator in: $(OBSERVATORIUM_NAMESPACE)"
	@oc kustomize config/operator/bases |sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g' |oc apply -f -
	@ for i in {1..12}; do oc -n $(OBSERVATORIUM_NAMESPACE) get pod -l control-plane=observatorium-operator -o name | grep "pod/observatorium-operator" && break || sleep 5; done


# Deploy Observatorium 
.PHONY: create/observatorium
create/observatorium:
ifeq ($(observatorium_env),dev)
create/observatorium: create/observatorium/dev
endif
# ifeq ($(observatorium_env),stage)
# create/observatorium: create/observatorium/stage
# endif
# ifeq ($(observatorium_env),prod)
# create/observatorium: create/observatorium/prod
# endif


.PHONY: create/observatorium/dev
create/observatorium/dev:
	@echo "create observatorium in: $(OBSERVATORIUM_NAMESPACE) for Dev Environment"
	cat config/observatorium/dev/thanos.yaml  > config/observatorium/dev/thanos.generated.yaml
	oc kustomize config/observatorium/dev |sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g;s/<hosturl>/$(OBSERVATORIUM_ROUTE_HOST)/g' |oc apply -f -

.PHONY: create/observatorium-up
create/observatorium-up:
	@echo "create observatorium-up job in: $(OBSERVATORIUM_NAMESPACE) for Dev Environment"
	oc kustomize config/observatorium-up/bases |sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g' |oc apply -f -

.PHONY: delete/observatorium-up
delete/observatorium-up:
	@echo "Delete observatorium-up job in: $(OBSERVATORIUM_NAMESPACE) for Dev Environment"
	oc delete job observatorium-up-metrics-tls -n $(OBSERVATORIUM_NAMESPACE)

.PHONY: create/cr
create/cr:
	@echo "create observatorium in: $(OBSERVATORIUM_NAMESPACE)"
	@sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g' ./observatorium.yaml | cat | oc apply -f -

# .PHONY: create/cr-multitenant
# create/cr-multitenant:
# 	@echo "create multitenant observatorium in: $(OBSERVATORIUM_NAMESPACE)"
# 	@sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g' ./observatorium-multitenant.yaml | cat | oc apply -f -

# .PHONY: create/dex-multitenant-secret
# create/dex-multitenant-secret:
# 	@echo "Updating dex config secret in:  $(DEX_NAMESPACE) for multitenancy"
# 	oc delete secret dex -n $(DEX_NAMESPACE);oc create secret generic dex --from-file=config.yaml=./config-multitenant.yaml -n $(DEX_NAMESPACE) --dry-run -o yaml|oc apply -f -
# 	oc delete pod -l app.kubernetes.io/name=dex -n $(DEX_NAMESPACE)

.PHONY: create/route
create/route:
	@echo "create route $(OBSERVATORIUM_ROUTE_HOST) in $(OBSERVATORIUM_NAMESPACE) namespace"
	@sed 's/<namespace>/$(OBSERVATORIUM_NAMESPACE)/g;s/<hosturl>/$(OBSERVATORIUM_ROUTE_HOST)/g' ./route.yaml | cat | oc apply -f -



#all-dev: create/dex create/minio create/crds create/operator create/observatorium
clean:
	@oc delete namespace $(DEX_NAMESPACE)
	@oc delete namespace $(MINIO_NAMESPACE)
	@oc delete namespace $(OBSERVATORIUM_NAMESPACE)
	@echo "Done uninstalling observatorium"


all: create/dex create/minio create/crds create/operator create/observatorium
	@echo "Done installing observatorium"