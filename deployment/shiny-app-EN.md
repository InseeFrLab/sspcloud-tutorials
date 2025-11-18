# Tutorial: Deploying a Shiny Application on the SSP Cloud

## Table of Contents

* [Tutorial: Deploying a Shiny Application on the SSP Cloud](#tutorial-deploying-a-shiny-application-on-the-ssp-cloud)

  * [Context](#context)
  * [Environment](#environment)
  * [Application Development](#application-development)

    * [Uploading Input Data to MinIO](#uploading-input-data-to-minio)
    * [Local Development Phase](#local-development-phase)
    * [Packaging](#packaging)
    * [Containerization](#containerization)
    * [Continuous Integration (CI)](#continuous-integration-ci)
  * [Application Deployment](#application-deployment)

    * [Creating a Helm Chart](#creating-a-helm-chart)
    * [Using S3 Data Storage with MinIO](#using-s3-data-storage-with-minio)
    * [Using a PostgreSQL Database](#using-a-postgresql-database)
    * [Deploying the Helm Chart](#deploying-the-helm-chart)
  * [Application Debugging/Maintenance](#application-debuggingmaintenance)

    * [Debugging Commands](#debugging-commands)
    * [Debugging Procedure](#debugging-procedure)
  * [Going Further](#going-further)

    * [ShinyProxy](#shinyproxy)

## Context

This tutorial documents the process of deploying an interactive web application on the [SSP Cloud](https://datalab.sspcloud.fr/home). The procedure is illustrated through the example of deploying an [R Shiny](https://shiny.rstudio.com/) application. Two GitHub repositories are used as templates:

* a [first repository](https://github.com/InseeFrLab/template-shiny-app) containing the source code of an illustrative Shiny application;
* a [second repository](https://github.com/InseeFrLab/template-shiny-deployment) containing the configuration files for deploying the application.

In practice, this tutorial can be easily adapted to deploy other interactive web applications using different frameworks such as [Flask](https://flask.palletsprojects.com/en/2.2.x/) or [Streamlit](https://streamlit.io/) in Python, for example.

## Environment

The application is deployed on the [SSP Cloud](https://datalab.sspcloud.fr/home), an instance of the [Onyxia](https://github.com/InseeFrLab/onyxia) project hosted at Insee. This instance is based on a [Kubernetes cluster](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/). Fundamentally, the deployment of application resources is therefore performed on Kubernetes. It is thus necessary to interact with the cluster to carry out the deployment.

To do this, work in the terminal of a VSCode service launched from the SSP Cloud. Important note: to deploy resources on the Kubernetes cluster, the VSCode service **must be launched with Admin rights on your Kubernetes namespace** (when initializing the service in the UI: Kubernetes tab → "Role" → select "admin"). This [tutorial](https://github.com/InseeFrLab/sspcloud-tutorials/blob/main/vscode-github/vscode-github.md) provides detailed guidance on using the VSCode service and interfacing it with Git and GitHub.

## Application Development

### Uploading Input Data to MinIO

The first step of the deployment process is to store input data so it can be accessed from the cluster. Data may be stored in your personal bucket or a shared bucket for collaborative projects.

The simplest approach is to upload data via a graphical interface—either the Datalab UI (in development) or directly the [MinIO console](https://minio-console.lab.sspcloud.fr/buckets). You may also interact with MinIO using R (RStudio), Python (JupyterLab or VSCode), or via the `mc` command-line client. The [Onyxia documentation](https://docs.sspcloud.fr/onyxia-guide/stockage-de-donnees) details these methods.

### Local Development Phase

Shiny applications are typically developed locally—either on a workstation or dedicated computing environment. Usually, this environment differs from the one used for deployment. For instance, development often happens on a Windows machine, while servers generally run Linux.

These differences require adjustments during deployment, which can sometimes be substantial. To minimize this risk, it is recommended to start development as early as possible in an environment close to production. On SSP Cloud, this may involve developing in an RStudio service launched from the [service catalog](https://datalab.sspcloud.fr/catalog/inseefrlab-helm-charts-datascience). While this service does not itself deploy Shiny apps to users, it allows development and testing in an environment consistent with deployment, making it possible to resolve issues and manage dependencies.

### Packaging

A recommended practice for simplifying deployment is to structure the project as an R package. This standardizes dependency management—explicitly listing dependencies rather than loading them via `library()` calls—and ensures a consistent structure for the deployment package.

The repository [shiny-app](https://github.com/InseeFrLab/template-shiny-app) provides a template Shiny application structured as an R package. Here is its structure:

```
shiny-app
├── Dockerfile
├── .gitlab-ci.yml
├── myshinyapp
│   ├── DESCRIPTION
│   ├── inst
│   │   └── app
│   │       ├── server.R
│   │       └── ui.R
│   ├── man
│   │   └── hello.Rd
│   ├── myshinyapp.Rproj
│   ├── NAMESPACE
│   └── R
│       ├── data.R
│       └── main.R
└── README.md
```

The Shiny application is integrated into an R package named `myshinyapp`. The `server` and `ui` components are located in a folder `inst/app`. Core functions are organized into `.R` modules inside the `R` directory. For example, `data.R` handles MinIO and PostgreSQL interactions, while `main.R` contains the function that launches the Shiny app.

The `DESCRIPTION` file must be completed carefully: it contains essential metadata and lists all package dependencies.

This minimal packaging method works well for simpler apps. For more complex applications, a production-oriented framework such as [golem](https://github.com/ThinkR-open/golem) is recommended. The book *Engineering Production-Grade Shiny Apps* provides an excellent overview.

### Containerization

To be deployable on Kubernetes, the application must be provided as a **Docker image**. Building a Docker image makes the application **portable** across environments.

The `Dockerfile` includes five main parts:

* **Base image**: `rocker/shiny` — containing R, the Shiny server, system libraries, and required dependencies.
* **Installation of system libraries** required by R packages.
* **Installation of the R package and dependencies**.
* **Exposing the application port** — typically no change needed.
* **Entrypoint** — the command that launches the container.

A dependency-installation workflow is illustrated using Mermaid in the original text.

### Continuous Integration (CI)

The file [`.github/workflows/ci.yaml`](https://github.com/InseeFrLab/template-shiny-app/blob/main/.github/workflows/ci.yaml) contains instructions executed each time code is modified in the Git repository. This CI process rebuilds the Docker image and pushes it to a registry.

This tutorial uses DockerHub, ideal for open-source projects. Steps include:

* create an authentication token on DockerHub,
* store credentials in GitHub secrets (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`),
* update the Docker image name accordingly.

## Application Deployment

### Creating a Helm Chart

Deploying the application requires a [Helm](https://helm.sh/) chart—a Kubernetes package defining the application's resources.

The [deployment repository](https://github.com/InseeFrLab/template-shiny-deployment) contains a Helm chart for the template application. Fork this repository to build your own chart.

The two key files are:

* **`Chart.yaml`** — metadata, version, parent chart dependencies (e.g., the [Insee Shiny chart](https://github.com/InseeFrLab/helm-charts/tree/master/charts/shiny)).
* **`values.yaml`** — all application-specific settings.

Modify:

* the image name,
* the image tag (using `latest` during development),
* the ingress hostname (must end in `*.lab.sspcloud.fr`).

### Using S3 Data Storage with MinIO

If your application requires external data stored on MinIO:

* enable `shiny.s3.enabled: true`,
* create a MinIO service account,
* create a Kubernetes Secret containing:

  * access key,
  * secret key,
  * endpoint,
  * default region.

Apply the Secret using:

```
kubectl apply -f secret.yaml
```

The environment variables are then automatically available inside the container.

### Using a PostgreSQL Database

If PostgreSQL is needed:

* enable `shiny.postgresql.enabled: true`,
* optionally customize the username, database name, and service hostname,
* create a Kubernetes Secret with PostgreSQL passwords.

As with MinIO, all values become environment variables available to the Shiny app.

### Deploying the Helm Chart

To deploy:

* clone your chart repository,
* run `helm dependency update`,
* install using:

```
helm install path_to_chart --generate-name
```

Check deployments via:

```
helm ls
```

## Application Debugging/Maintenance

Deployment may succeed while the application itself fails. Debugging typically involves checking Kubernetes resources—primarily **Pods**.

### Debugging Commands

List Pods:

```
kubectl get pods
```

Inspect a Pod:

```
kubectl describe pod pod_name
```

View logs:

```
kubectl logs pod_name
```

Open a shell inside the running container:

```
kubectl exec -it pod_name -- bash
```

### Debugging Procedure

#### If the error comes from the application:

* identify and fix the issue in the code,
* push changes,
* CI rebuilds the Docker image,
* update the image tag in `values.yaml`,
* run:

```
helm upgrade deployed_chart_name path_to_chart
```

#### If the error comes from the deployment configuration:

* inspect with `kubectl describe`,
* fix the chart,
* redeploy using `helm upgrade`.

## Going Further

### ShinyProxy

TODO

---

If you want, I can also:

✔ improve the English style for clarity and fluency
✔ shorten or restructure the text
✔ prepare a version suitable for documentation, slides, or training material

Just tell me!
