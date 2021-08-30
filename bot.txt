
import discord
from discord.ext import commands
from discord.ext import tasks
from discord import guild
from discord_slash import SlashCommand
import json
from time import sleep
from datetime import datetime
from twitwitwitwitch import *


intents = discord.Intents.default()
intents.members = True
bot = commands.Bot(command_prefix="!", help_command=None, intents=intents)
slash = SlashCommand(bot, sync_commands=True)

#####################################################################################
#                                   Twitch                                          #
#####################################################################################

@slash.slash(description="Ajouter un streamer")
async def twitch(ctx, pseudotwitch):

    with open('config.json') as config_file:
        config = json.load(config_file)

    if pseudotwitch in config['watchlist']:
        await ctx.send(f"Le streamer {pseudotwitch} à déja été ajouté")
    else:
        config["watchlist"].append(pseudotwitch)
        await ctx.send(f"Le streamer {pseudotwitch} à été ajouté")

    with open("config.json", 'w') as f:
        json.dump(config, f, indent=2)


@twitch.error
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"<@{ctx.message.author.id}> La commande est : {bot.command_prefix}twitch <pseudo>")


@bot.command()
async def listtwitch(ctx):

    with open('config.json') as config_file:
        config = json.load(config_file)


    embedvar = discord.Embed(title="Streamer List",
                             description=f"Twitch list", color=0x00ff00)

    for streams in config['watchlist']:
        embedvar.add_field(
            name=streams, value=f"{streams} est dans la twitch list", inline=False)

    await ctx.channel.send(embed=embedvar)

@slash.slash(description="Voir la liste des streamers")
async def twitchlist(ctx):

    with open('config.json') as config_file:
        config = json.load(config_file)


    embedvar = discord.Embed(title="Streamer List",
                             description=f"Twitch list", color=0x00ff00)

    for streams in config['watchlist']:
        embedvar.add_field(
            name=streams, value=f"{streams} est dans la twitch list", inline=False)

    await ctx.channel.send(embed=embedvar)

#####################################################################################
#                                   UnTwitch                                        #
#####################################################################################

@slash.slash(description="Supprimer un streamer")
async def untwitch(ctx, pseudotwitch: str):

    with open('config.json') as config_file:
        config = json.load(config_file)

    if pseudotwitch == "notsup":
        await ctx.send(f" Ne pas supprimer ce pseudo {pseudotwitch}")
    else:
        if pseudotwitch in config['watchlist']:
            config['watchlist'].remove(pseudotwitch)
            await ctx.send(f" Le streamer {pseudotwitch} à été supprimé")
        else:
            await ctx.send(f" Le streamer {pseudotwitch} n'est pas dans la list")

    with open("config.json", 'w') as f:
        json.dump(config, f, indent=2)


@twitch.error
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"<@{ctx.message.author.id}> La commande est : {bot.command_prefix}twitch <pseudo>")

#####################################################################################
#                                   Répétition                                      #
#####################################################################################

@tasks.loop(seconds=5)
async def streamers():

    with open('config.json') as config_file:
        config = json.load(config_file)

    def get_users(login_names):
        params = {
            "login": login_names
        }

        headers = {
            "Authorization": "Bearer {}".format(config["access_token"]),
            "Client-Id": config["client_id"]
        }

        reponse = requests.get(
            "https://api.twitch.tv/helix/users", params=params, headers=headers)
        return {entry["login"]: entry["id"] for entry in reponse.json()["data"]}
        # return reponse.json()

    def get_streams(users):
        params = {
            "user_id": users.values()
        }

        headers = {
            "Authorization": "Bearer {}".format(config["access_token"]),
            "Client-Id": config["client_id"]
        }
        reponse = requests.get(
            "https://api.twitch.tv/helix/streams", params=params, headers=headers)
        # return {entry["user_login"]: entry for entry in reponse.json()["data"]}
        return reponse.json()["data"]

    #####################################################################################
    #                                   Channel                                         #
    #####################################################################################
    channel = bot.get_channel(874336809163821146)

    now = datetime.now()

    current_time = now.strftime("%H:%M:%S")

    users = get_streams(get_users(config["watchlist"]))

    embedvar = discord.Embed(title="Streamer en live actuellement",
                             description=f"Dernière actuallisation à : {current_time}", color=0x00ff00)
    for user in users:

        user_name = user["user_name"]
        game_name = user["game_name"]
        title = user["title"]
        user_login = user["user_login"]
        link_live = "https://www.twitch.tv/" + user_login

        for guild in bot.guilds:
            for member in guild.members:

                server = guild
                membername = member.name
                memberstr = str(membername)
                memberlower = memberstr.lower()

                userstr = str(user_login)
                userlower = userstr.lower()

                #####################################################################################
                #                                   Role                                            #
                #####################################################################################
                role = discord.utils.get(server.roles, id=881933159028121630)

                if memberlower == userlower:
                    try:
                        await member.add_roles(role)
                    except:
                        print("Une erreur c'est produite votre grade est sûrement au dessus de celui du bot")
                else:
                    try:
                        await member.remove_roles(role)
                    except:
                        print("Une erreur c'est produite votre grade est sûrement au dessus de celui du bot")



        embedvar.add_field(
            name=user_name, value=f"```{game_name}``` {link_live}", inline=False)

    delet_message = await channel.history(limit=1).flatten()
    for message in delet_message:
        await message.delete()
    await channel.send(embed=embedvar)


@bot.event
async def on_member_remove(member):
    server = member.guild
    serverName = server.name
    pseudo = member.name

    with open('config.json') as config_file:
        config = json.load(config_file)

    if pseudo in config['watchlist']:
        print("Not")
    else:
        config["watchlist"].remove(pseudo)

    with open("config.json", 'w') as f:
        json.dump(config, f, indent=2)


@bot.event
async def on_ready():
    print("\nDémarrage du bot ....")
    print("Discord version : {}".format(discord.__version__))
    print("Connecté en : " + bot.user.name)
    print("Bot pret")
    streamers.start()
    await bot.change_presence(status=discord.Status.online)

bot.run("ODgxODk2MDAwNjA2ODMwNjEy.YSzfpw.iMgOs13sD07SnAlvPOlbkdJoaLI")
