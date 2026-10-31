# syntax=docker/dockerfile:1.7

# ============================================================
# Stage 1 — build: restore + publish on the full .NET 10 SDK
# ============================================================
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy only the csproj first so `dotnet restore` can be cached.
# While the project list is small this layer is cheap; it will
# only invalidate when CvBuilder.Api.csproj changes, not on
# every source edit.
COPY src/CvBuilder.Api/CvBuilder.Api.csproj src/CvBuilder.Api/
RUN dotnet restore src/CvBuilder.Api/CvBuilder.Api.csproj

# Now copy the rest of the API source and publish a Release build.
# UseAppHost=false drops the native launcher we won't use —
# the entrypoint runs `dotnet CvBuilder.Api.dll` directly.
COPY src/CvBuilder.Api/ src/CvBuilder.Api/
RUN dotnet publish src/CvBuilder.Api/CvBuilder.Api.csproj \
        --configuration Release \
        --no-restore \
        -p:UseAppHost=false \
        -o /app/publish

# ============================================================
# Stage 2 — runtime: ASP.NET 10 only, no SDK, no sources
# ============================================================
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

# Kestrel listens on 8080 inside the container.
# ASPNETCORE_ENVIRONMENT defaults to Production so OpenAPI
# and other dev-only middleware stay off.
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

COPY --from=build /app/publish ./

EXPOSE 8080

# The Microsoft .NET 8+ base images ship a non-root `app` user
# (UID exposed as $APP_UID). Switching to it drops root
# privileges for the running process.
USER $APP_UID

ENTRYPOINT ["dotnet", "CvBuilder.Api.dll"]
