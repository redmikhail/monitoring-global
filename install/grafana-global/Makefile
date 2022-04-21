SHELL=bash
export OBSERVATORIUM_APPS_URL := $(shell oc get --namespace openshift-ingress-operator ingresscontrollers/default --output jsonpath='{.status.domain}{"\n"}')
export DEX_TOKEN_URL := http://dex-dex.$(OBSERVATORIUM_APPS_URL)
export OBSERVATORIUM_ROUTE_HOST ?=observatorium-observatorium.$(OBSERVATORIUM_APPS_URL)
export GATEWAY_URL := https://$(OBSERVATORIUM_ROUTE_HOST)

#Temp
export GRAFANA_GLOBAL?=grafana-global
export GRAFANA_GLOBAL_NAMESPACE?=grafana-global




.PHONY: create/namespace
create/namespace:
	@echo "create namespace: $(GRAFANA_GLOBAL_NAMESPACE)"
	@sed 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g' ./namespace.yaml | cat | oc apply -f -



.PHONY: create/grafana/operator
create/grafana/operator:
	@echo "Install grafana operator: $(GRAFANA_GLOBAL)"
	@cat ./sub.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g' | \
		sed -e 's/<name>/$(GRAFANA_GLOBAL)/g' | \
		cat | oc apply -f -	
	@ echo "Waiting for Grafana CRD to be created"
	@ for i in {1..12}; do oc get crd grafanas.integreatly.org && break || sleep 5; done

.PHONY: create/grafana
create/grafana:
	@echo "Create grafana: $(GRAFANA_GLOBAL)"
	@cat ./grafana.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g' | \
		sed -e 's/<name>/$(GRAFANA_GLOBAL)/g' | \
		cat | oc apply -f -	
	
all: create/namespace create/grafana/operator create/grafana 

clean:
	@oc delete namespace $(GRAFANA_GLOBAL_NAMESPACE)
	@echo "Done uninstalling $(GRAFANA_GLOBAL)"
