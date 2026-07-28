# Stage de build: compilacao e publicacao movidas para o CodeBuild (buildspec.yml)
# conforme requisito do CI/CD - o build nao deve ocorrer dentro do Dockerfile
# FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
# WORKDIR /app
# COPY *.csproj ./
# RUN dotnet restore
# COPY . ./
# RUN dotnet publish -c Release -o out

# Stage de runtime: imagem final reduzida, apenas com o runtime do .NET 8
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# Define o diretorio de trabalho da aplicacao em runtime
WORKDIR /app

# Copia os artefatos ja publicados pelo CodeBuild
# O buildspec.yml executa: dotnet publish -o ./meusite/out
COPY ./out ./

# Configura o Kestrel para escutar na porta 8080 em todas as interfaces de rede
ENV ASPNETCORE_URLS=http://+:8080

# Documenta a porta utilizada pelo container
EXPOSE 8080

# Comando de inicializacao da aplicacao
ENTRYPOINT ["dotnet", "MeuSite.dll"]
