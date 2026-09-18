# Deploy a .NET API with Amazon ECS Express Mode

This repository follows [Getting Started with Amazon ECS Express Mode](https://dev.to/aws-builders/getting-started-with-amazon-ecs-express-mode-2pcd).
It deploys the `ApiSample` ASP.NET Core API to Amazon ECS Express Mode using:

- .NET 10
- GitHub Actions and GitHub Container Registry (GHCR)
- Terraform
- AWS ECS Express Mode

The API exposes a weather forecast endpoint and a `/health` endpoint that ECS
uses for health checks.

## Prerequisites

Install or configure the following before starting:

- .NET 10 SDK
- Terraform CLI
- Git and a GitHub repository
- An AWS account with permission to create IAM roles and ECS Express Gateway services
- AWS CLI credentials configured for the account

Verify the local tools:

```bash
dotnet --version
terraform version
aws sts get-caller-identity
```

The Terraform provider is configured for `ap-southeast-1` in
[`infra/main.tf`](infra/main.tf). Change the provider region and image
reference there if you want to deploy somewhere else.

## 1. Run the API locally

From the repository root, restore dependencies, build, and run the API:

```bash
dotnet restore
dotnet build
dotnet run --project ApiSample --launch-profile http
```

In another terminal, verify the health endpoint:

```bash
curl http://localhost:5007/health
```

The response should be:

```text
Healthy
```

You can also try the sample API at
`http://localhost:5007/weatherforecast`.

Press `Ctrl+C` in the running terminal when you are finished testing locally.

## 2. Publish the container image

The workflow in [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)
builds the container using the .NET SDK and publishes two tags to GHCR:

- `latest`
- the GitHub commit SHA

Push the repository to GitHub and use the `main` branch to trigger the
workflow:

```bash
git add .
git commit -m "Deploy API with ECS Express Mode"
git push origin main
```

You can also run the workflow manually from the **Actions** tab with
**Deploy > Run workflow**.

After the workflow succeeds, open the repository's **Packages** page and make
the `apisample` container package public. The Terraform configuration uses a
public GHCR image, so ECS must be able to pull it without package credentials.

The image configured by this repository is:

```text
ghcr.io/berviantoleo/aws-express-mode/apisample:latest
```

If you fork this repository, update the `image` value in
[`infra/main.tf`](infra/main.tf) to use your GitHub owner and repository.

## 3. Initialize Terraform

Change to the infrastructure directory and initialize the AWS provider:

```bash
cd infra
terraform init
```

Review the resources Terraform will create:

```bash
terraform plan
```

The configuration creates two IAM roles, their required AWS-managed policy
attachments, and one ECS Express Gateway service. The service is configured to
send traffic to container port `8080` and to check `/health`.

## 4. Deploy to ECS Express Mode

Apply the infrastructure:

```bash
terraform apply
```

Review the proposed changes and enter `yes` when prompted. When deployment
finishes, Terraform prints the service ingress path. ECS may need several
minutes to finish provisioning and start the task.

Check the URL returned by Terraform:

```bash
terraform output apisample_url
```

Append `/health` to the returned URL and open it in a browser or use `curl`:

```bash
curl "$(terraform output -raw apisample_url)/health"
```

The response should be `Healthy`. The sample API is available at the same host
under `/weatherforecast`.

## 5. Clean up AWS resources

When you are finished, remove the ECS Express service and IAM resources created
by this example:

```bash
terraform destroy
```

Enter `yes` when prompted. The ECS service declares dependencies on both IAM
roles so Terraform removes the service before deleting the roles.

The GHCR package is not managed by Terraform. Delete it separately from the
repository's **Packages** page if you no longer need it.

## Project layout

```text
ApiSample/                 ASP.NET Core API
.github/workflows/deploy.yml  Build and publish the GHCR image
infra/main.tf              ECS Express Mode Terraform configuration
```