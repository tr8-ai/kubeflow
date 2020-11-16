# jupyter server with custom image

```
python components/build_image.py --tf_version=2.1.0 --platform=cpu tf_notebook
docker push <repository> # of the above image (use: docker images)

# go to kubeflow ui running on http://127.0.0.1:8080
# create a new jupyter server, use the above repository
```
