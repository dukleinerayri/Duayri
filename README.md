import discord
from discord.ext import commands
from discord import app_commands
import os
import io
from flask import Flask
from threading import Thread

# ---- TOKEN ----
my_secret = os.environ['DISCORD']

# ---- INTENTS ----
intents = discord.Intents.default()
intents.message_content = True
intents.guilds = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)

# ---- SETTINGS ----
GUILD_ID = 1342949258956898344  # Server-ID
CATEGORY_IDS = {
    "Allgemein": 1343049435076231279,
    "Highrank": 1343049435076231279,
    "Partnerschaft": 1343049435076231279,
    "Leitung": 1343049435076231279
}
TICKET_ROLES = {
    "Allgemein": [1343045746005512263],
    "Highrank": [1343045746005512263],
    "Partnerschaft": [1343045746005512263],
    "Leitung": [1343045746005512263]
}
LOG_CHANNEL_ID = 1343200812418990180
MAX_TICKETS_PER_USER = 4  # Max. offene Tickets pro Benutzer

# ---- KEEP ALIVE für Repl.it ----
app = Flask('')

@app.route('/')
def home():
    return "Bot läuft 24/7!"

def run():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    t = Thread(target=run)
    t.start()

# ---- Ticket View ----
class TicketView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="Allgemeines Ticket", style=discord.ButtonStyle.primary, custom_id="ticket_allg")
    async def allg_button(self, interaction, button):
        await create_ticket(interaction, "Allgemein")

    @discord.ui.button(label="Bestellungsticket", style=discord.ButtonStyle.primary, custom_id="ticket_high")
    async def high_button(self, interaction, button):
        await create_ticket(interaction, "Highrank")

    @discord.ui.button(label="Beschwerde Ticket", style=discord.ButtonStyle.danger, custom_id="ticket_frac")
    async def frac_button(self, interaction, button):
        await create_ticket(interaction, "Partnerschaft")

    @discord.ui.button(label="Leitung Ticket", style=discord.ButtonStyle.secondary, custom_id="ticket_leitung")
    async def leitung_button(self, interaction, button):
        await create_ticket(interaction, "Leitung")

# ---- Close View ----
class CloseView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="Schließen", style=discord.ButtonStyle.danger, custom_id="close_ticket")
    async def close(self, interaction, button):
        await interaction.response.send_message(
            embed=discord.Embed(
                title="Wollen Sie das Ticket wirklich schließen?",
                description="Bitte wähle unten eine Option.",
                color=discord.Color.red()
            ),
            view=ConfirmCloseView(),
            ephemeral=True
        )

class ConfirmCloseView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="Geöffnet lassen", style=discord.ButtonStyle.success, custom_id="keep_open")
    async def keep_open(self, interaction, button):
        await interaction.response.edit_message(
            embed=discord.Embed(title="Ticket bleibt offen ✅", color=discord.Color.green()),
            view=None
        )

    @discord.ui.button(label="Schließen", style=discord.ButtonStyle.danger, custom_id="confirm_close")
    async def confirm_close(self, interaction, button):
        guild = interaction.guild
        channel = interaction.channel
        log_channel = guild.get_channel(LOG_CHANNEL_ID)

        # Interaktion bestätigen, bevor Channel gelöscht wird
        await interaction.response.send_message("🔒 Das Ticket wird nun geschlossen...", ephemeral=True)

        # Transkript erstellen
        transcript_text = ""
        async for msg in channel.history(limit=None, oldest_first=True):
            timestamp = msg.created_at.strftime("%Y-%m-%d %H:%M:%S")
            line = f"[{timestamp}] {msg.author}: {msg.content}\n"
            transcript_text += line

        transcript_file = discord.File(io.BytesIO(transcript_text.encode()), filename=f"transcript-{channel.name}.txt")

        if log_channel:
            embed = discord.Embed(
                title="📕 Ticket geschlossen",
                description=f"Ticket **{channel.name}** wurde von {interaction.user.mention} geschlossen.",
                color=discord.Color.red()
            )
            await log_channel.send(embed=embed, file=transcript_file)

        # Channel löschen
        await channel.delete()

# ---- Ticket Erstellung ----
async def create_ticket(interaction: discord.Interaction, category_name: str):
    guild = interaction.guild

    # Max. 4 Tickets pro Benutzer prüfen
    user_tickets = [c for c in guild.text_channels if c.name.startswith(f"ticket-{interaction.user.name.lower()}")]
    if len(user_tickets) >= MAX_TICKETS_PER_USER:
        await interaction.response.send_message(
            f"❌ Du kannst maximal {MAX_TICKETS_PER_USER} Tickets gleichzeitig offen haben.",
            ephemeral=True
        )
        return

    category = guild.get_channel(CATEGORY_IDS[category_name])

    overwrites = {
        guild.default_role: discord.PermissionOverwrite(view_channel=False),
        interaction.user: discord.PermissionOverwrite(view_channel=True, send_messages=True, attach_files=True, embed_links=True)
    }

    for role_id in TICKET_ROLES[category_name]:
        role = guild.get_role(role_id)
        if role:
            overwrites[role] = discord.PermissionOverwrite(view_channel=True, send_messages=True, manage_messages=True)

    ticket_channel = await guild.create_text_channel(
        name=f"ticket-{interaction.user.name.lower()}-{len(user_tickets)+1}",
        category=category,
        overwrites=overwrites
    )

    embed = discord.Embed(
        title="📩 Dein Ticket",
        description=f"||<@&{TICKET_ROLES[category_name][0]}>||\n\nEin Teammitglied wird sich so schnell wie möglich um dein Ticket kümmern.\nBitte vermeide es, Teammitglieder zu pingen.",
        color=discord.Color.blue()
    )

    await ticket_channel.send(embed=embed, view=CloseView())
    await interaction.response.send_message(
        f"✅ Dein Ticket wurde erstellt: {ticket_channel.mention}",
        ephemeral=True
    )

# ---- /ticketpanel Command ----
@bot.tree.command(name="ticketpanel", description="Erstellt das Ticket-Panel mit den Buttons")
async def ticketpanel(interaction: discord.Interaction):
    if interaction.guild.id != GUILD_ID:
        await interaction.response.send_message("❌ Dieses Ticket-System ist nur auf dem vorgesehenen Server verfügbar.", ephemeral=True)
        return

    await interaction.response.send_message("✅ Ticket-Panel wird erstellt...", ephemeral=True)
    channel = interaction.channel

    # Info-Embed
    info_embed = discord.Embed(
        title="🎫 **TICKET SYSTEM**",
        description=(
            "Haben Sie Anliegen oder Fragen?\n\n"
            "__**Allgemeines Ticket:**__\n・ Allgemeine Fragen\n\n"
            "__**Beschwerde Ticket:**__\n・ Beschwerden gegen Staffler\n・ Probleme bei Designs\n\n"
            "__**Leitungsticket:**__\n・ Allgemeine Leitungs-Frage\n・ Partnerschaftsbeantragung"
        ),
        color=discord.Color.blurple()
    )
    info_embed.set_footer(text="Bitte wähle das passende Ticket unten aus!")

    # Bild-Embed
    embeds = [info_embed]
    image_path = "ticketbild.png"
    if os.path.isfile(image_path):
        with open(image_path, "rb") as f:
            file = discord.File(f, filename="ticketbild.png")
            image_embed = discord.Embed(color=discord.Color.blurple())
            image_embed.set_image(url="attachment://ticketbild.png")
            embeds.append(image_embed)
            await channel.send(embeds=embeds, file=file, view=TicketView())
    else:
        image_url = "https://i.imgur.com/deinbild.png"
        image_embed = discord.Embed(color=discord.Color.blurple())
        image_embed.set_image(url=image_url)
        embeds.append(image_embed)
        await channel.send(embeds=embeds, view=TicketView())

# ---- on_ready ----
@bot.event
async def on_ready():
    await bot.wait_until_ready()
    guild = discord.Object(id=GUILD_ID)
    bot.tree.copy_global_to(guild=guild)
    await bot.tree.sync(guild=guild)
    print(f"✅ Slash-Commands synchronisiert für Guild-ID: {GUILD_ID}")
    print(f"🤖 Eingeloggt als {bot.user}")

# ---- START ----
keep_alive()  # Repl.it 24/7 Webserver starten
bot.run(my_secret)
# Duayri
