# Guide complet d'implémentation d'un serveur MCP pour Claude Desktop et Cursor AI

## Table des matières

1. [Introduction au Model Context Protocol (MCP)](#introduction-au-model-context-protocol-mcp)
2. [Architecture et composants MCP](#architecture-et-composants-mcp)
3. [Prérequis et installation](#prérequis-et-installation)
4. [Création d'un serveur MCP en Python](#création-dun-serveur-mcp-en-python)
5. [Étude de cas : Serveur météo](#étude-de-cas--serveur-météo)
6. [Intégration avec Claude Desktop](#intégration-avec-claude-desktop)
7. [Intégration avec Cursor AI](#intégration-avec-cursor-ai)
8. [Fonctionnalités avancées](#fonctionnalités-avancées)
9. [Débogage et résolution des problèmes](#débogage-et-résolution-des-problèmes)
10. [Meilleures pratiques](#meilleures-pratiques)
11. [Ressources supplémentaires](#ressources-supplémentaires)

## Introduction au Model Context Protocol (MCP)

Le Model Context Protocol (MCP) est un protocole ouvert qui standardise la façon dont les applications fournissent du contexte aux modèles de langage (LLM). On peut comparer MCP à un port USB-C pour les applications d'IA : tout comme USB-C offre une manière standardisée de connecter vos appareils à divers périphériques et accessoires, MCP offre une méthode standardisée pour connecter les modèles d'IA à différentes sources de données et outils.

### Pourquoi MCP est important

Avant MCP, les utilisateurs devaient copier-coller du code dans une interface textuelle pour interagir avec les LLM. Cette approche s'est rapidement avérée insuffisante, conduisant au développement d'intégrations personnalisées pour un meilleur chargement de contexte. Cependant, ces solutions étaient fragmentées et nécessitaient un développement individuel. MCP résout ce problème en offrant un protocole universel pour une interaction efficace de l'IA avec des ressources locales et distantes.

### Avantages du MCP

- **Standardisation** : Interface unifiée pour tous les modèles et outils
- **Interopérabilité** : Les serveurs MCP peuvent être utilisés par n'importe quel client compatible
- **Extensibilité** : Facile à étendre avec de nouvelles fonctionnalités
- **Sécurité** : Contrôle précis sur l'accès aux ressources

## Architecture et composants MCP

L'écosystème MCP se compose de quatre éléments principaux :

### 1. Hôtes

Les hôtes sont des applications LLM (comme Claude Desktop ou Cursor) qui initient les connexions aux serveurs MCP. Ils découvrent les capacités des serveurs et planifient comment les utiliser pour résoudre les problèmes des utilisateurs.

### 2. Clients

Les clients maintiennent des connexions 1:1 avec les serveurs, à l'intérieur de l'application hôte. Ils gèrent la communication entre l'hôte et les serveurs.

### 3. Serveurs

Les serveurs MCP fournissent trois types principaux de capacités aux clients :
- **Ressources** : Données de type fichier pouvant être lues par les clients (comme des réponses d'API ou des contenus de fichiers)
- **Outils** : Fonctions pouvant être appelées par le LLM (avec l'approbation de l'utilisateur)
- **Invites (Prompts)** : Modèles pré-écrits qui aident les utilisateurs à accomplir des tâches spécifiques

### 4. Transports

Les transports définissent comment les clients et serveurs communiquent. MCP prend en charge plusieurs méthodes de transport, notamment :
- **stdio** : Communication via l'entrée/sortie standard
- **sse** : Communication via Server-Sent Events

## Prérequis et installation

### Exigences système

- Ordinateur Mac ou Windows
- Python 3.10 ou supérieur installé
- Dernière version de `uv` installée (gestionnaire de packages Python)

### Installation de l'environnement

Pour installer `uv` sous MacOS/Linux :

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Création et configuration d'un nouveau projet :

```bash
# Créer un nouveau répertoire pour notre projet
uv init mcp_server
cd mcp_server

# Créer un environnement virtuel et l'activer
uv venv
source .venv/bin/activate  # Sous Windows: .venv\Scripts\activate

# Installer les dépendances
uv add "mcp[cli]" httpx

# Créer notre fichier serveur
touch server.py
```

## Création d'un serveur MCP en Python

### Structure de base d'un serveur MCP

Un serveur MCP comporte généralement les éléments suivants :

1. **Initialisation** : Configuration du serveur avec un nom
2. **Définition des outils** : Création de fonctions décorées avec `@mcp.tool()`
3. **Exécution** : Démarrage du serveur avec une méthode de transport

Voici un exemple de structure minimale :

```python
from mcp.server.fastmcp import FastMCP

# Initialiser le serveur MCP
mcp = FastMCP("mon_serveur")

@mcp.tool()
async def mon_outil(param1: str, param2: int) -> str:
    """Description de ce que fait l'outil.
    
    Args:
        param1: Description du premier paramètre
        param2: Description du deuxième paramètre
    """
    # Logique de l'outil
    return "Résultat"

if __name__ == "__main__":
    # Démarrer le serveur (stdio ou sse)
    mcp.run(transport="stdio")
```

### Types de données et validation

MCP prend en charge les types suivants pour les paramètres et les valeurs de retour :
- Types primitifs : `str`, `int`, `float`, `bool`
- Types composés : `list`, `dict`
- Types personnalisés via Pydantic

## Étude de cas : Serveur météo

Analysons le serveur météo implémenté dans le fichier `weather.py` :

```python
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# Initialiser le serveur FastMCP
mcp = FastMCP("weather")

# Constantes
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"

async def make_nws_request(url: str) -> dict[str, Any] | None:
    """Effectue une requête à l'API NWS avec gestion d'erreur appropriée."""
    headers = {"User-Agent": USER_AGENT, "Accept": "application/geo+json"}
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None

def format_alert(feature: dict) -> str:
    """Formate une alerte en chaîne lisible."""
    props = feature["properties"]
    return f"""
Event: {props.get('event', 'Unknown')}
Area: {props.get('areaDesc', 'Unknown')}
Severity: {props.get('severity', 'Unknown')}
Description: {props.get('description', 'No description available')}
Instructions: {props.get('instruction', 'No specific instructions provided')}
"""

@mcp.tool()
async def get_alerts(state: str) -> str:
    """Obtient les alertes météo pour un état américain.

    Args:
        state: Code à deux lettres d'un état américain (ex. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "Impossible de récupérer les alertes ou aucune alerte trouvée."

    if not data["features"]:
        return "Aucune alerte active pour cet état."

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """Obtient les prévisions météo pour un emplacement.

    Args:
        latitude: Latitude de l'emplacement
        longitude: Longitude de l'emplacement
    """
    # D'abord obtenir le point de grille des prévisions
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "Impossible de récupérer les données de prévision pour cet emplacement."

    # Obtenir l'URL des prévisions depuis la réponse du point
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "Impossible de récupérer les prévisions détaillées."

    # Formater les périodes en prévisions lisibles
    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:  # Montrer seulement les 5 prochaines périodes
        forecast = f"""
{period['name']}:
Temperature: {period['temperature']}°{period['temperatureUnit']}
Wind: {period['windSpeed']} {period['windDirection']}
Forecast: {period['detailedForecast']}
"""
        forecasts.append(forecast)

    return "\n---\n".join(forecasts)

if __name__ == "__main__":
    # Initialiser et exécuter le serveur
    mcp.run(transport="stdio")
```

### Points clés de l'implémentation

1. **Initialisation** : Le serveur est initialisé avec le nom "weather"
2. **Fonctions utilitaires** : `make_nws_request` et `format_alert` sont des fonctions d'aide
3. **Outils** : Deux outils sont définis avec `@mcp.tool()` :
   - `get_alerts` : Récupère les alertes météo pour un état américain
   - `get_forecast` : Récupère les prévisions météo pour une localisation géographique
4. **Transport** : Le serveur utilise le transport "stdio"

## Intégration avec Claude Desktop

Claude Desktop est l'une des applications qui prend en charge le protocole MCP.

### Configuration dans Claude Desktop

1. Ouvrez Claude Desktop
2. Accédez aux paramètres (icône d'engrenage)
3. Sélectionnez l'onglet "MCP"
4. Cliquez sur "Ajouter un serveur"
5. Entrez la commande pour démarrer votre serveur MCP, par exemple :
   ```
   python /chemin/vers/weather.py
   ```
6. Sauvegardez les paramètres

### Test du serveur

Dans Claude Desktop, vous pouvez maintenant utiliser les capacités de votre serveur MCP en posant des questions comme :

- "Quelles sont les alertes météo actuelles pour CA ?"
- "Quelle est la prévision météo pour la latitude 34.05 et la longitude -118.24 ?"

Claude interpretera automatiquement ces questions et utilisera les outils appropriés de votre serveur MCP.

## Intégration avec Cursor AI

Cursor AI est un éditeur de code intelligent qui prend également en charge le protocole MCP.

### Configuration dans Cursor AI

1. Ouvrez Cursor AI
2. Accédez aux paramètres
3. Sélectionnez "Model Context Protocol" dans la section "Context"
4. Cliquez sur "Add Server"
5. Configurez votre serveur MCP avec la commande appropriée, par exemple :
   ```
   python /chemin/vers/weather.py
   ```
6. Sauvegardez les paramètres

### Utilisation dans Cursor AI

Dans Cursor AI, vous pouvez utiliser les outils MCP en interagissant avec l'agent IA. Vous pouvez poser des questions ou demander des informations qui nécessitent l'utilisation de vos outils MCP.

Par exemple, dans la fenêtre de chat de Cursor, vous pourriez demander :
- "Quelles sont les alertes météo pour NY ?"
- "Peux-tu me donner la prévision météo pour San Francisco (latitude 37.77, longitude -122.41) ?"

## Fonctionnalités avancées

### Ressources MCP

En plus des outils, MCP permet de fournir des ressources, qui sont des contenus lisibles par le LLM :

```python
@mcp.resource()
async def get_documentation() -> str:
    """Renvoie la documentation de l'API."""
    with open("docs.md", "r") as f:
        return f.read()
```

### Prompts MCP

MCP prend en charge les prompts, qui sont des modèles textuels paramétrables :

```python
@mcp.prompt()
async def weather_report_template(location: str, date: str) -> str:
    """Template pour générer un rapport météo.
    
    Args:
        location: Nom de l'emplacement
        date: Date du rapport
    """
    return f"""
Rapport météo pour {location} - {date}

[Insérez ici les détails météo pour {location}]

Préparé par: Service météo MCP
"""
```

### Contrôle d'accès et authentification

Pour les scénarios plus sécurisés, vous pouvez implémenter un contrôle d'accès personnalisé :

```python
from mcp.server.fastmcp import FastMCP, AuthContext

async def custom_auth(auth_context: AuthContext) -> bool:
    # Vérifiez l'authentification de l'utilisateur
    return auth_context.user_id in AUTHORIZED_USERS

mcp = FastMCP("secure_server", auth_handler=custom_auth)
```

## Débogage et résolution des problèmes

### Journalisation

Ajoutez la journalisation pour faciliter le débogage :

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("mcp_server")

@mcp.tool()
async def my_tool(param: str) -> str:
    logger.debug(f"Tool called with param: {param}")
    # ...
```

### Problèmes courants et solutions

1. **Erreur "Failed to connect to server"**
   - Vérifiez que le serveur est en cours d'exécution
   - Assurez-vous que le chemin spécifié est correct

2. **Le serveur démarre mais l'hôte ne le détecte pas**
   - Vérifiez que vous utilisez le bon transport (stdio/sse)
   - Redémarrez l'application hôte

3. **Les outils sont détectés mais renvoient des erreurs**
   - Vérifiez la validité des types de paramètres
   - Assurez-vous que les APIs externes fonctionnent

## Meilleures pratiques

### Structure du code

- Séparez la logique métier de la définition des outils MCP
- Créez des fonctions d'aide pour éviter la duplication de code
- Implémentez une gestion d'erreur robuste

### Documentation

- Fournissez des descriptions détaillées pour chaque outil
- Documentez clairement les paramètres et leurs types
- Incluez des exemples d'utilisation

### Performance

- Utilisez des connexions asynchrones pour les opérations réseau
- Mettez en cache les résultats fréquemment demandés
- Limitez la taille des données renvoyées

## Ressources supplémentaires

- [Documentation officielle MCP](https://modelcontextprotocol.io/)
- [SDK Python MCP](https://github.com/modelcontextprotocol/python-sdk)
- [SDK TypeScript MCP](https://github.com/modelcontextprotocol/typescript-sdk)
- [Exemples de serveurs MCP](https://modelcontextprotocol.io/example-servers)
- [Communauté Discord MCP](https://discord.gg/modelcontextprotocol)

---

## Conclusion

Le Model Context Protocol (MCP) représente une avancée majeure dans la façon dont nous interagissons avec les modèles de langage. En créant un serveur MCP et en l'intégrant à des applications comme Claude Desktop et Cursor AI, vous pouvez étendre considérablement les capacités de ces outils d'IA et créer des expériences plus riches et plus utiles pour les utilisateurs.

Ce guide vous a fourni les bases pour commencer à développer vos propres serveurs MCP. Vous êtes maintenant prêt à explorer des cas d'utilisation plus avancés et à contribuer à l'écosystème MCP en croissance.

N'hésitez pas à expérimenter avec différents types d'outils, de ressources et de prompts pour découvrir tout le potentiel du protocole MCP ! 