##############################################################################
# Stage 0: Common CBL-Mariner core base
##############################################################################
FROM mcr.microsoft.com/cbl-mariner/base/core:2.0 AS base

RUN tdnf update -y \
 && tdnf install -y \
      ca-certificates curl tar gzip unzip git icu tzdata shadow-utils \
      glibc libgcc libstdc++ openssl-libs zlib \
 && tdnf clean all

##############################################################################
# Stage 1: Full .NET SDK Installer (includes host/fxr)
##############################################################################
FROM base AS sdk-installer
ARG DOTNET_SDK_VERSION=8.0.411

RUN tdnf install -y tar \
 && tdnf clean all \
 && curl -fsSL \
      https://dotnetcli.azureedge.net/dotnet/Sdk/${DOTNET_SDK_VERSION}/dotnet-sdk-${DOTNET_SDK_VERSION}-linux-x64.tar.gz \
      -o dotnet-sdk.tar.gz \
 && mkdir -p /dotnet \
 && tar -oxzf dotnet-sdk.tar.gz -C /dotnet \
 && rm dotnet-sdk.tar.gz

##############################################################################
# Stage 2: ASP.NET Core Runtime Installer
##############################################################################
FROM base AS aspnetcore-installer
ARG ASPNETCORE_VERSION=8.0.17

RUN tdnf install -y tar \
 && tdnf clean all \
 && curl -fsSL \
      https://builds.dotnet.microsoft.com/dotnet/aspnetcore/Runtime/${ASPNETCORE_VERSION}/aspnetcore-runtime-${ASPNETCORE_VERSION}-linux-x64.tar.gz \
      -o aspnetcore.tar.gz \
 && mkdir -p /dotnet \
 && tar -oxzf aspnetcore.tar.gz -C /dotnet \
 && rm aspnetcore.tar.gz

##############################################################################
# Stage 3: .NET Runtime Installer
##############################################################################
FROM base AS runtime-installer
ARG RUNTIME_VERSION=8.0.17

RUN tdnf install -y tar \
 && tdnf clean all \
 && curl -fsSL \
      https://builds.dotnet.microsoft.com/dotnet/Runtime/${RUNTIME_VERSION}/dotnet-runtime-${RUNTIME_VERSION}-linux-x64.tar.gz \
      -o dotnet-runtime.tar.gz \
 && mkdir -p /dotnet \
 && tar -oxzf dotnet-runtime.tar.gz -C /dotnet \
 && rm dotnet-runtime.tar.gz

##############################################################################
# Stage 4: SDK “Base” for building the Functions Host
##############################################################################
FROM base AS sdk-base

# Copy in the full SDK (including host/fxr)
COPY --from=sdk-installer /dotnet /usr/share/dotnet

RUN ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet

##############################################################################
# Stage 5: Build the Azure Functions Host (self-contained)
##############################################################################
FROM sdk-base AS host-builder
ARG HOST_VERSION=4.1037.0

# Install only what dotnet publish needs
RUN tdnf install -y icu tzdata unzip curl ca-certificates \
 && tdnf clean all

# 1) Bump System.Drawing.Common → 4.7.2 (fixes CVE-2021-24112)
# 2) Build self-contained host
RUN git clone --depth 1 --branch v${HOST_VERSION} \
      https://github.com/Azure/azure-functions-host /src/azure-functions-host \
 && cd /src/azure-functions-host \
 \
 && sed -i \
    's|<PackageReference Include="System.Drawing.Common" Version=".*" />|<PackageReference Include="System.Drawing.Common" Version="4.7.2" />|' \
    src/WebJobs.Script.WebHost/WebJobs.Script.WebHost.csproj \
 \
 && HOST_COMMIT=$(git rev-list -1 HEAD) \
 && dotnet publish src/WebJobs.Script.WebHost/WebJobs.Script.WebHost.csproj \
      -c Release \
      --output /azure-functions-host \
      --runtime linux-x64 \
      --self-contained \
      /p:BuildNumber=$(echo ${HOST_VERSION} | cut -d'.' -f3) \
      /p:CommitHash=$HOST_COMMIT \
 \
 && rm -rf /src/azure-functions-host /root/.nuget /root/.local

##############################################################################
# Stage 6: Final runtime image (pure CBL-Mariner core)
##############################################################################
FROM base AS final
ARG HOST_VERSION

# Minimal runtime deps
RUN tdnf install -y dnf glibc-devel icu tzdata \
                  ca-certificates openssl-libs zlib \
 && tdnf clean all

# 1) Copy .NET runtime bits
COPY --from=runtime-installer /dotnet /usr/share/dotnet
# 2) Copy ASP.NET shared framework
COPY --from=aspnetcore-installer /dotnet/shared/Microsoft.AspNetCore.App \
                                  /usr/share/dotnet/shared/Microsoft.AspNetCore.App

RUN ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet

# Copy self-contained Functions Host
COPY --from=host-builder /azure-functions-host /azure-functions-host


# Strip out **all** workers & extension bundles (removes all those Java/Maven CVEs)
RUN rm -rf /azure-functions-host/workers \
&& rm -rf /FunctionExtensionBundles

# Prepare function app folder
RUN mkdir -p /home/site/wwwroot
WORKDIR /home/site/wwwroot

ENV \
  AzureWebJobsScriptRoot=/home/site/wwwroot \
  HOME=/home \
  FUNCTIONS_WORKER_RUNTIME=dotnet-isolated \
  DOTNET_USE_POLLING_FILE_WATCHER=true \
  HOST_VERSION=${HOST_VERSION} \
  ASPNETCORE_CONTENTROOT=/azure-functions-host \
  AzureWebJobsFeatureFlags=EnableWorkerIndexing \
  ASPNETCORE_URLS=http://+:80

EXPOSE 80
ENTRYPOINT ["/azure-functions-host/Microsoft.Azure.WebJobs.Script.WebHost"]
