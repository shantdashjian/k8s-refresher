# KubeView Instllation Steps

 1. Clone the GitHub repo of KubeView locally:
```
> git clone https://github.com/benc-uk/kubeview.git
```
 2. Navigate to the `charts` directory:
```
> cd kubeview/charts
```
 3. Make a local copy of `example-values.yaml`:
```
> cp example-values.yaml myvalues.yaml
```
 4. Open `myvalues.yaml` in your text editor and edit the following lines to prevent exposing the service using a load balancer:
 ```
loadBalancer:
  enabled: false
```
 5. Install KubeView using Helm. If you don't have Helm installed yet, install it [here](https://helm.sh/docs/intro/install/):
```
> helm install kubeview ./kubeview -f myvalues.yaml
```
 6. Expose the service using port forwarding:
```
> kubectl port-forward service/kubeview 8000:80
```
 7. In your browser, navigate to `localhost:8000`:
 

[![kubernetes cluster visualized in KubeView][1]][1]

  [1]: https://i.sstatic.net/8blkskTK.png