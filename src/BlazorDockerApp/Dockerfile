# syntax=docker/dockerfile:1

# ---- Build stage ----
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy project file and restore dependencies (cached layer)
COPY ["BlazorDockerApp.csproj", "./"]
RUN dotnet restore "BlazorDockerApp.csproj"

# Copy the rest of the sources and publish
COPY . .
RUN dotnet publish "BlazorDockerApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

# ---- Runtime stage ----
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
EXPOSE 8080

ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "BlazorDockerApp.dll"]
