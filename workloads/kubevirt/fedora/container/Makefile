VER := 6

image := quay.io/nirsof/dr-demo:fedora-$(VER)

container: fedora.qcow2
	podman build -t $(image) .

push:
	podman push $(image)

fedora.qcow2:
	curl -L -o $@ https://download.fedoraproject.org/pub/fedora/linux/releases/40/Cloud/x86_64/images/Fedora-Cloud-Base-Generic.x86_64-40-1.14.qcow2

clean:
	rm -f *.qcow2
