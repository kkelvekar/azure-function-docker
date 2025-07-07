# Use the Azure Functions .NET 8.0 isolated worker base image
FROM mcr.microsoft.com/azure-functions/dotnet-isolated:4-dotnet-isolated8.0-mariner AS base

# Create the default function app folder (host won't error if empty)
RUN mkdir -p /home/site/wwwroot

# Expose the Functions default port
EXPOSE 80
