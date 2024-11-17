# Bot-Lookup
import discord
from discord.ext import commands
import os
import requests
import logging
import json
from difflib import get_close_matches
import traceback
import subprocess
import uuid
import codecs
from discord import Embed, Button
import random
import asyncio
import shodan
import ipaddress
import re
from discord import PartialEmoji, Emoji

# Créer un verrou
import asyncio

LONGUEUR_MINIMALE_COMMANDE = 3

command_lock = asyncio.Lock()

CATEGORIE_ID_AUTORISEE = 1304918234813698050
authorized_category_id = 1304918234813698050

blacklist_role_id = 1304918234813698050

CATEGORY_ID = 1304918234813698050

blacklist_file_path = 'blacklist.txt'

snusbase_api = "https://api-experimental.snusbase.com"
snusbase_auth = "UR API SNUSBASE"

TOKEN = 'ur token bot'

search_cache = {}

blacklisted_users = [""]

logging.basicConfig(filename='logs.txt', level=logging.INFO)

intent = discord.Intents.all()
bot = commands.Bot(command_prefix='+', intents=intent)
bot.remove_command('help')

def no_dms():
    async def predicate(ctx):
        if ctx.guild is None:
            raise commands.CommandError("Cette commande ne peut pas être utilisée en message privé.")
        return True
    return commands.check(predicate)

def restricted_to_category():
    async def predicate(ctx):
        if ctx.channel.category_id == authorized_category_id:
            return True
        else:
            await ctx.send("Cette commande n'est pas autorisée dans cette catégorie.")
            return False
    return commands.check(predicate)

# Définition des décorateurs pour bloquer l'utilisation des commandes avec un seul, deux ou trois caractères
def block_short_commands():
    async def predicate(ctx):
        command_content = ctx.message.content[len(bot.command_prefix):]
        if len(command_content) <= 3:
            await ctx.send("Commande invalide.")
            return False
        return True
    return commands.check(predicate)


# Commande exemple avec vérification de la longueur du paramètre
@bot.command()
@block_short_commands()
async def example_command(ctx, *, param):
    await ctx.reply(f"Commande exécutée avec succès avec le paramètre : {param}")

# Commande unique avec vérification de la liste noire des utilisateurs
@bot.command(name='unique_command')
@block_short_commands()
async def unique_command(ctx):
    user_id = str(ctx.author.id)
    if user_id in blacklisted_users:
        await ctx.reply("Tu joues à quoi frero ? Mdrrrr")
        return
    await ctx.reply("Commande exécutée avec succès !")

# Commande restreinte à une catégorie spécifique
@bot.command()
@restricted_to_category()
@block_short_commands()
async def restricted_command(ctx):
    await ctx.send("Cette commande est autorisée dans la catégorie spécifiée.")

# Gestion des commandes exécutées en message privé
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.NoPrivateMessage):
        await ctx.send("Cette commande ne peut pas être utilisée en message privé.")

# Gestion des commandes exécutées en dehors de la catégorie spécifiée
@bot.event
async def on_command(ctx):
    if ctx.guild and ctx.guild.get_channel(ctx.channel.id).category_id != authorized_category_id:
        await ctx.send("Cette commande ne peut pas être utilisée dans cette catégorie.")

# Gestion des messages
# Créer un décorateur pour vérifier la longueur de la commande
# Définition du décorateur pour vérifier la longueur de la commande

# Exemple d'utilisation du décorateur
# Fonction pour enregistrer l'activité
def log_activity(message):
    logging.info(message)

# Gestion des erreurs de commande
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        command_content = ctx.message.content.split()[0][1:]
        suggestions = get_close_matches(command_content, bot.all_commands.keys(), n=1, cutoff=0.5)
        if suggestions:
            suggested_command = suggestions[0]
            await ctx.reply(f"Commande introuvable ou inexistante. Voici une commande similaire : ``+{suggested_command}``")
        else:
            await ctx.reply("Commande introuvable ou inexistante. Aucune suggestion disponible.")

@bot.event
async def on_ready():
    print(f'{bot.user} est connecté à Discord.')


@bot.command()
@restricted_to_category()
async def ip(ctx, ip_address: str):
    await ctx.reply("https://discord.gg/jvUQWxgW") 
    
    category_id = 1304918234813698050
    if ctx.channel.category_id == category_id:
        api_url = f'http://ip-api.com/json/{ip_address}?fields=status,message,continent,continentCode,country,countryCode,region,regionName,city,district,zip,lat,lon,timezone,offset,currency,isp,org,as,asname,reverse,mobile,proxy,hosting,query'

        # Effectuer la requête vers l'API ip
        response = requests.get(api_url)
        
        # Vérifier si la requête a réussi
        if response.status_code == 200:
            data = response.json()
            
            # Créer une embed pour afficher les informations
            embed = discord.Embed(
                title=f'Informations ip pour l\'adresse IP {ip_address}',
                color=0x00ff00
            )
            
            # Ajouter les champs pour chaque information disponible
            for key, value in data.items():
                embed.add_field(name=key, value=str(value), inline=False)
            
            # Envoyer l'embed sur Discord
            await ctx.reply(embed=embed)
        else:
            await ctx.reply(f'Erreur lors de la récupération des informations pour l\'adresse IP {ip_address}.')
    else:
        await ctx.reply(
            "🚫 Cette commande ne peut être exécutée que dans les canaux de texte de la catégorie spécifiée."
        )

@bot.command()
@restricted_to_category()
async def phone(ctx, phone_number):
    await ctx.reply("UR discord") 
    
    category_id = [1304918234813698050]
    if ctx.channel.category_id in category_id:
        try:
            phone_number = phone_number.lstrip('+')
            if not phone_number.isdigit():
                await ctx.reply("Veuillez fournir un numéro de téléphone valide.")
                return
            
            # NE PAS TOUCHER c'est le lien vers l'api sans sa la cmd marche plus !!
            api_url = f'http://apilayer.net/api/validate?access_key=1cf3dd77bf1ad472dedd792814d7da7c&number={phone_number}'
            response = requests.get(api_url)
            data = response.json()
            
            if 'valid' in data:
                if data['valid']:
                    embed = discord.Embed(
                        title=f"Phone Informations - {phone_number}",
                        color=0x000000
                    )
                    embed.add_field(name="Numéro", value=data['number'], inline=False)
                    embed.add_field(name="Format local", value=data['local_format'], inline=False)
                    embed.add_field(name="Format international", value=data['international_format'], inline=False)
                    embed.add_field(name="Préfixe du pays", value=data['country_prefix'], inline=False)
                    embed.add_field(name="Code pays", value=data['country_code'], inline=False)
                    embed.add_field(name="Nom du pays", value=data['country_name'], inline=False)
                    embed.add_field(name="Localisation", value=data['location'], inline=False)
                    embed.add_field(name="Opérateur", value=data['carrier'], inline=False)
                    embed.add_field(name="Type de ligne", value=data['line_type'], inline=False)
                    await ctx.reply(embed=embed)
                else:
                    await ctx.reply(f"Le numéro de téléphone {phone_number} n'est pas valide.")
            else:
                await ctx.reply(f"La réponse de l'API n'a pas le format attendu.")
        except Exception as e:
            await ctx.reply(f"Une erreur s'est produite lors de la recherche des informations pour le numéro de téléphone {phone_number}.\n{str(e)}")
    else:
        await ctx.reply(
            "🚫 Cette commande ne peut être exécutée que dans les canaux de texte des catégories spécifiées."
        )


@bot.command()
@restricted_to_category()
async def info(ctx, user_id: int):
    await ctx.reply("ur discord") 
    
    blacklist_ids = []
    if user_id in blacklist_ids:
        return await ctx.reply("Cet utilisateur n'est pas disponible.")
    
    category_id = [1304918234813698050]
    if ctx.channel.category_id not in category_id:
        return await ctx.reply(
            "🚫 Cette commande ne peut être exécutée que dans les canaux de texte des catégories spécifiées."
        )
    
    try:
        # Obtenir l'objet User à partir de l'ID
        user = await bot.fetch_user(user_id)
        
        # Créer une embed pour afficher les informations de l'utilisateur
        embed = discord.Embed(
            title=f'Informations utilisateur pour ID {user.id}',
            color=0x00ff00
        )
        
        # Ajouter les champs pour les informations de base de l'utilisateur
        embed.add_field(name='Nom', value=user.name, inline=True)
        embed.add_field(name='Discriminateur', value=user.discriminator, inline=True)
        embed.add_field(name='ID', value=user.id, inline=True)
        embed.set_thumbnail(url=user.avatar.url if user.avatar else user.default_avatar.url)
        
        # Vérifier si l'utilisateur est membre d'un serveur partagé avec le bot
        member = ctx.guild.get_member(user.id)
        if member:
            # Ajouter la bannière et la biographie si disponibles
            if member.banner:
                embed.set_image(url=member.banner)
            if member.public_flags:
                bio_flags = member.public_flags
                bio = bio_flags.all()
                if bio:
                    embed.add_field(name='Biographie', value=bio, inline=False)
                
            # Ajouter d'autres informations disponibles sur l'utilisateur
            if member.activities:
                embed.add_field(name='Activités', value=', '.join([activity.name for activity in member.activities]), inline=False)
            if member.colour:
                embed.add_field(name='Couleur du rôle', value=member.colour, inline=False)
            if member.created_at:
                embed.add_field(name='Créé le', value=member.created_at.strftime("%Y-%m-%d %H:%M:%S"), inline=False)
            if member.joined_at:
                embed.add_field(name='Rejoint le', value=member.joined_at.strftime("%Y-%m-%d %H:%M:%S"), inline=False)
            if member.guild_permissions:
                embed.add_field(name='Permissions sur le serveur', value=', '.join([perm for perm, value in member.guild_permissions if value]), inline=False)
        
        # Envoyer l'embed sur Discord
        await ctx.reply(embed=embed)
    except discord.NotFound:
        await ctx.reply(f'Utilisateur avec l\'ID {user_id} non trouvé.')
    except Exception as e:
        print(f'Erreur lors de la récupération des informations utilisateur : {e}')
        await ctx.reply("Une erreur s'est produite lors de la récupération des informations utilisateur.")



@bot.command()
@restricted_to_category()
async def github(ctx, github_id: str):
    await ctx.reply("J'ai Trouvé 🤓") 
    
    category_id = [1304918234813698050]
    if ctx.channel.category_id in category_id:
        try:
            api_url = f'https://api.github.com/users/{github_id}'
            
            # Effectuer la requête vers l'API GitHub
            response = requests.get(api_url)
            
            # Vérifier si la requête a réussi
            if response.status_code == 200:
                data = response.json()
                
                # Créer une embed pour afficher les informations GitHub
                embed = discord.Embed(
                    title=f'Informations pour le compte GitHub de {github_id}',
                    color=0x6f42c1  # Couleur violette (couleur GitHub)
                )
                
                # Ajouter les champs pour chaque information disponible
                embed.add_field(name='Nom d\'utilisateur', value=data['login'], inline=True)
                embed.add_field(name='Nom complet', value=data.get('name', 'Non spécifié'), inline=True)
                embed.add_field(name='Bio', value=data.get('bio', 'Aucune bio disponible'), inline=False)
                embed.add_field(name='Followers', value=data['followers'], inline=True)
                embed.add_field(name='Following', value=data['following'], inline=True)
                embed.add_field(name='Repos publics', value=data['public_repos'], inline=True)
                embed.add_field(name='Création du compte', value=data['created_at'], inline=False)
                
                # Ajouter l'avatar de l'utilisateur GitHub à l'embed
                embed.set_thumbnail(url=data['avatar_url'])
                
                # Envoyer l'embed sur Discord
                await ctx.reply(embed=embed)
            else:
                await ctx.reply(f'Erreur lors de la récupération des informations pour le compte GitHub de {github_id}')
        except Exception as e:
            await ctx.reply(f"Une erreur s'est produite : {e}")
    else:
        await ctx.reply(
            "🚫 Cette commande ne peut être exécutée que dans les canaux de texte des catégories spécifiées."
        )


@bot.command()
@restricted_to_category()
@block_short_commands()
@no_dms()
async def Search(ctx, term: str):
    if term in search_cache:
        await ctx.send(search_cache[term])
        return
    
    await ctx.reply("Recherche en cours... ``Délais estimé entre 2 et 5 minutes.``")

    async with ctx.typing():
        files_to_search = []
        search_directory = r'ur db bro'

        for root, dirs, files in os.walk(search_directory):
            for file in files:
                if file.endswith(".txt"):
                    file_path = os.path.join(root, file)
                    try:
                        with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                            content = f.read()
                            files_to_search.append(content)
                    except Exception as e:
                        print(f"Erreur de lecture du fichier {file}: {e}")

        output = ''
        timeout = 30
        max_lines = 50
        start_time = asyncio.get_event_loop().time()
        line_count = 0

        for file_content in files_to_search:
            if line_count >= max_lines:
                break
            if asyncio.get_event_loop().time() - start_time > timeout:
                break

            for line in file_content.splitlines():
                if term.lower() in line.lower():
                    output += line + "\n"
                    line_count += 1
                    print(f"Ligne trouvée: {line}")

        message_header = '''\
███╗   ██╗███████╗██████╗ ██████╗     ██╗      ██████╗  ██████╗ ██╗  ██╗██╗   ██╗██████╗ 
████╗  ██║██╔════╝██╔══██╗██╔══██╗    ██║     ██╔═══██╗██╔═══██╗██║ ██╔╝██║   ██║██╔══██╗
██╔██╗ ██║█████╗  ██████╔╝██║  ██║    ██║     ██║   ██║██║   ██║█████╔╝ ██║   ██║██████╔╝
██║╚██╗██║██╔══╝  ██╔══██╗██║  ██║    ██║     ██║   ██║██║   ██║██╔═██╗ ██║   ██║██╔═══╝ 
██║ ╚████║███████╗██║  ██║██████╔╝    ███████╗╚██████╔╝╚██████╔╝██║  ██╗╚██████╔╝██║     
╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝╚═════╝     ╚══════╝ ╚═════╝  ╚═════╝ ╚═╝  ╚═╝ ╚═════╝ ╚═╝     
'''

        if output:
            complete_message = f"```\n{message_header}\n{output}\n```"
            search_cache[term] = complete_message
            await ctx.send(complete_message)
        else:
            await ctx.reply("Aucun résultat trouvé.")


@bot.command(name='help')
@restricted_to_category()
async def help_command(ctx):
    # Enregistrez l'activité dans les logs
    log_activity(f"{ctx.author.name} ({ctx.author.id}) a demandé de l'aide.")

    # Vérifiez si la commande est exécutée dans le salon spécifié
    allowed_category_ids = [1304918234813698050]
    
    if ctx.channel.category_id in allowed_category_ids:
        # Création de l'embed avec un titre stylisé
        embed = discord.Embed(
            title="Voici Le Nerd Bot🤓", 
            description="[Fonctionnalité ?👇]",
            color=0xFF0000
        )
        
        # Ajout de champs pour chaque commande avec des emojis correspondants
        embed.add_field(
            name="**+snusbase🤓**",
            value="*Sort des informations telles que les emails, mots de passe, noms, utilisateurs, adresses IP, hash, mot de passes d'un nom ou d'une adresse email spécifiés.\nUtilisation : `+snusbase <Nom/Email/Username/IP/Passwords/Hash>`*",
            inline=False
        )
        embed.add_field(
            name="**+ip🤓**",
            value="*Sort toutes les informations liées à une adresse IP spécifiée.\nUtilisation : `+ip <Adresse IP>`*",
            inline=False
        )
        embed.add_field(
            name="**+search🤓**",
            value="*Recherche les informations associées à une ID Discord ou FiveM spécifiée.\nUtilisation : `+search <ID>`*",
            inline=False
        )
        embed.add_field(
            name="**+minecraft🤓**",
            value="*Sort toutes les informations liées à un nom d'utilisateur Minecraft spécifié.\nUtilisation : `+minecraft <Nom d'utilisateur Minecraft>`*",
            inline=False
        )
        embed.add_field(
            name="**+github🤓**",
            value="*Sort toutes les informations liées à un nom d'utilisateur GitHub spécifié.\nUtilisation : `+github <Nom d'utilisateur GitHub>`*",
            inline=False
        )
        embed.add_field(
            name="**+phone🤓**",
            value="*Fournit des informations sur une ligne téléphonique spécifiée.\nUtilisation : `+phone <Numéro de téléphone>`*",
            inline=False
        )

        # Ajout du pied de page avec le lien
        embed.set_footer(text="discord.gg/nerdlookup by str3ong.")

        # Envoyer l'embed
        await ctx.reply(embed=embed)
    else:
        # Si la commande est exécutée dans un autre salon, avertissez l'utilisateur
        await ctx.reply("🚫 Cette commande ne peut être utilisée que dans les canaux de texte des catégories spécifiées.")





def send_request(url, body=None):
    headers = {
        'Auth': snusbase_auth,
        'Content-Type': 'application/json',
    }
    method = 'POST' if body else 'GET'
    data = json.dumps(body) if body else None
    full_url = snusbase_api + '/' + url
    response = requests.request(method, full_url, headers=headers, data=data)
    return response.json()

# Commande Snusbase
@bot.command(name='snusbase')
async def snusbase_command(ctx, term):
    # Vérifier si la commande est utilisée dans un salon de la catégorie autorisée
    if ctx.guild and ctx.channel.category_id != authorized_category_id:
        return await ctx.reply("Cette commande n'est autorisée que dans une catégorie spécifique.")

    # Chemin du fichier de résultat
    result_file_path = os.path.join('snusbase', f'{term}.txt')

    try:
        if os.path.exists(result_file_path):
            await ctx.reply("Envoi du résultat ...")
            await ctx.reply(file=discord.File(result_file_path))
            return

        # Rechercher dans Snusbase
        search_response = send_request('data/search', {
            'terms': [term],
            'types': ['username', 'email', 'lastip', 'hash', 'password', 'name'],
            'wildcard': False,
        })

        os.makedirs('snusbase', exist_ok=True)
        with open(result_file_path, 'w') as file:
            file.write(json.dumps(search_response, indent=2))

        await ctx.reply("Envoi du résultat ...")
        await ctx.reply(file=discord.File(result_file_path))

    except Exception as e:
        print(f"Erreur lors de la recherche Snusbase : {e}")
        await ctx.reply("Une erreur s'est produite lors de la recherche dans Snusbase. Veuillez réessayer plus tard.")

bot.run(TOKEN)
