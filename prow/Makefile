apply-config:
	kubectl create configmap plugins \
		--from-file=plugins.yaml=plugins.yaml --dry-run=client -o yaml \
		| kubectl apply -f -
	kubectl create configmap config \
		--from-file=config.yaml=config.yaml --dry-run=client -o yaml \
		| kubectl apply -f -
	kubectl create configmap label-config \
		--from-file=labels.yaml=label_sync/labels.yaml --dry-run=client -o yaml \
		| kubectl apply -f -

deploy:
	kubectl apply -k manifest/

deploy-check:
	kubectl apply -k manifest/ --dry-run=client -o yaml
