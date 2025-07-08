##############################################################################
# Stage 1: .NET SDK Installer
##############################################################################
FROM mcr.microsoft.com/dotnet/aspnet:8.0.17-cbl-mariner2.0-amd64 AS sdk-installer
ARG DOTNET_SDK_VERSION=8.0.411

# Install tar and unpack only the SDK bits we need
RUN tdnf install -y tar \
 && tdnf clean all \
 && curl -fSL \
      https://dotnetcli.azureedge.net/dotnet/Sdk/${DOTNET_SDK_VERSION}/dotnet-sdk-${DOTNET_SDK_VERSION}-linux-x64.tar.gz \
      -o /tmp/dotnet-sdk.tar.gz \
 && mkdir -p /dotnet \
 && tar -oxzf /tmp/dotnet-sdk.tar.gz -C /dotnet \
      ./packs ./sdk ./sdk-manifests ./templates ./LICENSE.txt ./ThirdPartyNotices.txt \
 && rm /tmp/dotnet-sdk.tar.gz

##############################################################################
# Stage 2: SDK Image (for building the Functions Host)
##############################################################################
FROM mcr.microsoft.com/dotnet/aspnet:8.0.17-cbl-mariner2.0-amd64 AS sdk-base
ARG DOTNET_SDK_VERSION=8.0.411

ENV \
    DOTNET_GENERATE_ASPNET_CERTIFICATE=false \
    DOTNET_NOLOGO=true \
    DOTNET_SDK_VERSION=${DOTNET_SDK_VERSION} \
    DOTNET_USE_POLLING_FILE_WATCHER=true \
    NUGET_XMLDOC_MODE=skip \
    POWERSHELL_DISTRIBUTION_CHANNEL=PSDocker-DotnetSDK-CBL-Mariner-2.0

# Install Git & runtime prerequisites, hook up dotnet CLI
RUN tdnf install -y git libatomic_ops tar \
 && tdnf clean all \
 && ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet

COPY --from=sdk-installer /dotnet /usr/share/dotnet
RUN dotnet help

##############################################################################
# Stage 3: Build the Azure Functions Host
##############################################################################
FROM sdk-base AS host-builder
ARG HOST_VERSION=4.1037.0

# Install certificates, ICU and other libs so `dotnet publish` runs correctly
RUN tdnf install -y ca-certificates icu tzdata unzip curl \
 && tdnf clean all

# Clone the exact Functions Host version and publish it self-contained
RUN git clone --depth 1 --branch v${HOST_VERSION} \
      https://github.com/Azure/azure-functions-host /src/azure-functions-host \
 && cd /src/azure-functions-host \
 && HOST_COMMIT=$(git rev-list -1 HEAD) \
 && dotnet publish src/WebJobs.Script.WebHost/WebJobs.Script.WebHost.csproj \
      -c Release \
      --output /azure-functions-host \
      --runtime linux-x64 \
      --self-contained \
      /p:BuildNumber=$(echo ${HOST_VERSION} | cut -d'.' -f3) \
      /p:CommitHash=$HOST_COMMIT \
 && rm -rf /src/azure-functions-host /root/.nuget /root/.local

##############################################################################
# Stage 4: ASP.NET Core Shared Framework Base
##############################################################################
FROM mcr.microsoft.com/dotnet/aspnet:8.0.17-cbl-mariner2.0-amd64 AS aspnet-base

##############################################################################
# Stage 5: Final Image
##############################################################################
FROM mcr.microsoft.com/dotnet/runtime:8.0.17-cbl-mariner2.0-amd64 AS final
ARG HOST_VERSION

# Install DNF and glibc-devel for proper runtime support of the host
RUN tdnf install -y dnf glibc-devel \
 && tdnf clean all

ENV \
  AzureWebJobsScriptRoot=/home/site/wwwroot \
  HOME=/home \
  FUNCTIONS_WORKER_RUNTIME=dotnet-isolated \
  DOTNET_USE_POLLING_FILE_WATCHER=true \
  HOST_VERSION=${HOST_VERSION} \
  ASPNETCORE_CONTENTROOT=/azure-functions-host \
  AzureWebJobsFeatureFlags=EnableWorkerIndexing \
  ASPNETCORE_URLS=http://+:80

# Copy in the compiled Functions Host
COPY --from=host-builder /azure-functions-host /azure-functions-host

# Bring in the ASP.NET shared framework so HTTP/Kestrel work
COPY --from=aspnet-base /usr/share/dotnet/shared/Microsoft.AspNetCore.App /usr/share/dotnet/shared/Microsoft.AspNetCore.App

# Prepare function app folder
RUN mkdir -p /home/site/wwwroot
WORKDIR /home/site/wwwroot

# Launch the self-contained host
ENTRYPOINT ["/azure-functions-host/Microsoft.Azure.WebJobs.Script.WebHost"]
