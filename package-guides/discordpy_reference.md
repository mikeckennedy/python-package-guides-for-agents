# discord.py Comprehensive Reference

> Generated from the discord.py v2.8.0a source code repository. This reference covers the full
> public API with class signatures, method parameters, event names, and common patterns — verified
> against the actual implementation.

---

## Table of Contents

1. [Public API (Top-Level Imports)](#1-public-api-top-level-imports)
2. [Client Class](#2-client-class)
3. [Intents](#3-intents)
4. [Bot (ext.commands)](#4-bot-extcommands)
5. [Prefix Commands](#5-prefix-commands)
6. [Cogs](#6-cogs)
7. [Application Commands (Slash Commands)](#7-application-commands-slash-commands)
8. [Interactions](#8-interactions)
9. [UI Components (Views, Buttons, Selects, Modals)](#9-ui-components-views-buttons-selects-modals)
10. [Messages](#10-messages)
11. [Embeds](#11-embeds)
12. [Guilds (Servers)](#12-guilds-servers)
13. [Channels](#13-channels)
14. [Members and Users](#14-members-and-users)
15. [Roles](#15-roles)
16. [Permissions](#16-permissions)
17. [Threads](#17-threads)
18. [Webhooks](#18-webhooks)
19. [Voice](#19-voice)
20. [Enums](#20-enums)
21. [Errors and Exceptions](#21-errors-and-exceptions)
22. [Events](#22-events)
23. [Tasks (ext.tasks)](#23-tasks-exttasks)
24. [Converters (ext.commands)](#24-converters-extcommands)
25. [Utility Classes (File, Colour, AllowedMentions)](#25-utility-classes-file-colour-allowedmentions)
26. [Common Patterns](#26-common-patterns)

---

## 1. Public API (Top-Level Imports)

Everything needed is importable directly from `discord` or its submodules:

```python
import discord
from discord import (
    # Core classes
    Client, AutoShardedClient,
    User, ClientUser, Member,
    Guild, GuildPreview,
    Message, PartialMessage, Attachment,
    Role, Emoji, PartialEmoji, Reaction,
    Invite, Template, Widget,
    Webhook, SyncWebhook,

    # Channel types
    TextChannel, VoiceChannel, StageChannel,
    DMChannel, GroupChannel, CategoryChannel,
    ForumChannel, Thread,

    # Rich content
    Embed, File, Colour, Color,  # Colour and Color are aliases

    # Configuration
    Intents, MemberCacheFlags,
    Permissions, PermissionOverwrite,
    AllowedMentions,

    # Activities & presence
    Activity, Game, Streaming, Spotify, CustomActivity,
    Status,

    # Flag types
    MessageFlags, ApplicationFlags, PublicUserFlags,

    # Components & interactions
    Interaction, InteractionResponse,
    SelectOption,

    # Monetization
    Entitlement, SKU,

    # Application info
    AppInfo, Team,

    # Stickers
    Sticker, GuildSticker, StandardSticker, StickerPack,

    # Misc
    Object, VoiceClient, VoiceProtocol,
)

# Submodules
from discord import ui           # Views, Buttons, Selects, Modals, TextInputs
from discord import app_commands # Slash commands, context menus, CommandTree
from discord import utils        # Utility functions (get, find, utcnow, etc.)
from discord.ext import commands # Bot, Cog, prefix commands framework
from discord.ext import tasks    # Background task loop helper
```

---

## 2. Client Class

The base client for connecting to Discord. Use `commands.Bot` instead if you need prefix commands.

### Constructor

```python
client = discord.Client(
    *,
    intents: discord.Intents,               # REQUIRED — controls which events are received
    max_messages: Optional[int] = 1000,     # Message cache size (None to disable)
    proxy: Optional[str] = None,            # HTTP proxy URL
    proxy_auth: Optional[aiohttp.BasicAuth] = None,
    shard_id: Optional[int] = None,
    shard_count: Optional[int] = None,
    application_id: Optional[int] = None,
    member_cache_flags: MemberCacheFlags = MemberCacheFlags.all(),
    chunk_guilds_at_startup: bool = True,   # Request full member list on startup
    status: Optional[Status] = None,        # Initial bot status
    activity: Optional[BaseActivity] = None,# Initial bot activity
    allowed_mentions: Optional[AllowedMentions] = None,
    heartbeat_timeout: float = 60.0,
    guild_ready_timeout: float = 2.0,
    assume_unsync_clock: bool = True,
    enable_debug_events: bool = False,
    max_ratelimit_timeout: Optional[float] = None,
    connector: Optional[aiohttp.BaseConnector] = None,
)
```

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `user` | `Optional[ClientUser]` | The connected bot user |
| `guilds` | `Sequence[Guild]` | All guilds the bot is in |
| `emojis` | `Sequence[Emoji]` | All cached emojis |
| `cached_messages` | `Sequence[Message]` | Message cache |
| `private_channels` | `Sequence[PrivateChannel]` | Open DM channels |
| `voice_clients` | `List[VoiceProtocol]` | Active voice connections |
| `application_id` | `Optional[int]` | Bot's application ID |
| `latency` | `float` | WebSocket latency in seconds |

### Key Methods

```python
# Connection lifecycle
await client.login(token: str)              # Authenticate (HTTP only)
await client.connect(*, reconnect=True)     # Open WebSocket
await client.start(token: str)              # login() + connect()
client.run(token: str)                      # Blocking: start() + event loop
await client.close()                        # Disconnect and clean up
await client.wait_until_ready()             # Block until READY event
client.is_ready() -> bool
client.is_closed() -> bool

# Async setup hook — override for async initialization
async def setup_hook(self) -> None: ...

# Wait for a specific event
await client.wait_for(
    'message',                              # Event name (without 'on_' prefix)
    check=lambda m: m.author.id == 123,     # Filter predicate
    timeout=60.0                            # Seconds before TimeoutError
)

# Cache getters
client.get_channel(id) -> Optional[Channel]
client.get_guild(id) -> Optional[Guild]
client.get_user(id) -> Optional[User]
client.get_emoji(id) -> Optional[Emoji]
client.get_all_channels() -> Iterator[GuildChannel]
client.get_all_members() -> Iterator[Member]

# Fetch from API (bypasses cache)
await client.fetch_user(id) -> User
await client.fetch_guild(id) -> Guild
await client.fetch_channel(id) -> Channel

# Status
await client.change_presence(
    status=discord.Status.online,
    activity=discord.Game(name='a game')
)
```

---

## 3. Intents

Intents control which gateway events the bot receives. **`intents` is required** in the `Client` and `Bot` constructors.

```python
# Common patterns
intents = discord.Intents.default()         # Most intents (excludes privileged ones)
intents = discord.Intents.all()             # Everything (requires portal toggles)
intents = discord.Intents.none()            # Nothing — build up manually

# Enable specific intents
intents = discord.Intents.default()
intents.message_content = True              # PRIVILEGED — read message.content
intents.members = True                      # PRIVILEGED — member join/leave/update
intents.presences = True                    # PRIVILEGED — presence updates
```

### Available Intents

| Intent | Privileged | Events Gated |
|--------|-----------|--------------|
| `guilds` | No | Guild create/update/delete, channel events |
| `members` | **Yes** | Member join/remove/update, thread member sync |
| `bans` | No | Ban add/remove |
| `emojis` | No | Emoji/sticker update |
| `expressions` | No | Emoji/sticker/soundboard update |
| `integrations` | No | Integration events |
| `webhooks` | No | Webhook update |
| `invites` | No | Invite create/delete |
| `voice_states` | No | Voice state update |
| `presences` | **Yes** | Presence update |
| `messages` | No | Message create/update/delete (guild + DM) |
| `guild_messages` | No | Message events in guilds only |
| `dm_messages` | No | Message events in DMs only |
| `message_content` | **Yes** | `message.content`, `embeds`, `attachments`, `components` |
| `reactions` | No | Reaction add/remove |
| `typing` | No | Typing indicator |
| `guild_scheduled_events` | No | Scheduled events |
| `auto_moderation` | No | AutoMod rule/action events |
| `polls` | No | Poll vote add/remove |

> **Note:** Privileged intents must be enabled in the Discord Developer Portal under **Bot > Privileged Gateway Intents**.

---

## 4. Bot (ext.commands)

`commands.Bot` extends `Client` with prefix command support, cog management, and extension loading.

```python
from discord.ext import commands

bot = commands.Bot(
    command_prefix='!',                     # str, list of str, or callable
    *,
    intents=discord.Intents.default(),      # REQUIRED
    strip_after_prefix: bool = False,
    case_insensitive: bool = False,
    owner_id: Optional[int] = None,
    owner_ids: Optional[Collection[int]] = None,
    # ... all Client parameters also accepted
)
```

### Prefix Helpers

```python
# Respond to @mention as prefix
bot = commands.Bot(command_prefix=commands.when_mentioned, intents=intents)

# Respond to @mention OR '!'
bot = commands.Bot(command_prefix=commands.when_mentioned_or('!'), intents=intents)

# Dynamic prefix via callable
async def get_prefix(bot, message):
    return ['!', '?']
bot = commands.Bot(command_prefix=get_prefix, intents=intents)
```

### Extension/Cog Management

```python
# Extensions — Python modules with an async setup(bot) function
await bot.load_extension('cogs.moderation')
await bot.unload_extension('cogs.moderation')
await bot.reload_extension('cogs.moderation')

# Cog management
await bot.add_cog(MyCog(bot))
await bot.remove_cog('MyCog')
bot.get_cog('MyCog') -> Optional[Cog]
bot.cogs -> Mapping[str, Cog]
```

### App Command Syncing

```python
# Bot has a built-in CommandTree accessible via bot.tree
@bot.event
async def setup_hook():
    await bot.tree.sync()                        # Sync globally
    await bot.tree.sync(guild=discord.Object(id=GUILD_ID))  # Sync to guild
```

---

## 5. Prefix Commands

### Basic Command

```python
@bot.command(
    name='ping',                            # Command name (defaults to function name)
    aliases=['p', 'latency'],               # Alternative triggers
    help='Check bot latency',               # Long help text
    brief='Ping the bot',                   # Short help text
    hidden=False,                           # Hide from help
    enabled=True,                           # Enable/disable command
)
async def ping(ctx: commands.Context):
    await ctx.send(f'Pong! {round(bot.latency * 1000)}ms')
```

### Command with Arguments

```python
@bot.command()
async def greet(ctx, member: discord.Member, *, reason: str):
    """Greet a member. Arguments after * are keyword-only (rest of message)."""
    await ctx.send(f'Hello {member.mention}! {reason}')

# Optional arguments
@bot.command()
async def info(ctx, member: discord.Member = None):
    member = member or ctx.author
    await ctx.send(f'{member.display_name} joined at {member.joined_at}')
```

### Command Groups

```python
@bot.group(invoke_without_command=True)
async def config(ctx):
    await ctx.send('Use a subcommand: `config set` or `config get`')

@config.command()
async def set(ctx, key: str, value: str):
    await ctx.send(f'Set {key} = {value}')

@config.command()
async def get(ctx, key: str):
    await ctx.send(f'{key} = ...')
```

### Checks and Cooldowns

```python
@bot.command()
@commands.has_permissions(manage_messages=True)
async def purge(ctx, count: int):
    await ctx.channel.purge(limit=count)

@bot.command()
@commands.has_role('Moderator')
async def warn(ctx, member: discord.Member):
    ...

@bot.command()
@commands.cooldown(rate=1, per=60.0, type=commands.BucketType.user)
async def daily(ctx):
    ...

# Other checks
@commands.guild_only()          # Only in servers
@commands.dm_only()             # Only in DMs
@commands.is_owner()            # Only bot owner
@commands.bot_has_permissions(send_messages=True)
@commands.has_any_role('Admin', 'Moderator')
```

### Context Object

```python
class commands.Context:
    bot: Bot
    message: Message
    author: Union[User, Member]
    channel: Union[TextChannel, DMChannel, Thread]
    guild: Optional[Guild]
    command: Command
    prefix: str
    invoked_with: str

    # Sending
    await ctx.send(content, *, embed, file, files, view, ephemeral, ...)
    await ctx.reply(content, ...)           # Reply to invoking message
    await ctx.typing()                      # "Bot is typing..." indicator
    await ctx.defer()                       # Defer response for interactions
```

---

## 6. Cogs

Cogs organize related commands, listeners, and state into reusable classes.

```python
class Moderation(commands.Cog):
    """Moderation commands for server management."""

    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command()
    @commands.has_permissions(kick_members=True)
    async def kick(self, ctx, member: discord.Member, *, reason=None):
        await member.kick(reason=reason)
        await ctx.send(f'Kicked {member.display_name}')

    @commands.Cog.listener()
    async def on_member_join(self, member: discord.Member):
        channel = member.guild.system_channel
        if channel:
            await channel.send(f'Welcome {member.mention}!')

    # App commands work in cogs too
    @app_commands.command(description='Ban a user')
    @app_commands.checks.has_permissions(ban_members=True)
    async def ban(self, interaction: discord.Interaction, member: discord.Member):
        await member.ban()
        await interaction.response.send_message(f'Banned {member}')

# Extension setup function — required for bot.load_extension()
async def setup(bot: commands.Bot):
    await bot.add_cog(Moderation(bot))
```

### GroupCog (Slash Command Group as a Cog)

```python
class Config(commands.GroupCog, name='config', description='Server configuration'):
    def __init__(self, bot):
        self.bot = bot

    @app_commands.command(name='set')
    async def config_set(self, interaction, key: str, value: str):
        await interaction.response.send_message(f'Set {key}={value}')
```

---

## 7. Application Commands (Slash Commands)

### CommandTree

The `CommandTree` manages registration and dispatch of application commands.

```python
# Standalone Client usage
client = discord.Client(intents=intents)
tree = app_commands.CommandTree(client)

# Bot already has one: bot.tree
```

### Slash Commands

```python
@tree.command(name='hello', description='Say hello')
async def hello(interaction: discord.Interaction):
    await interaction.response.send_message('Hello!')

# With parameters and descriptions
@tree.command(description='Echo a message')
@app_commands.describe(
    message='The message to echo',
    ephemeral='Whether only you can see the response'
)
async def echo(interaction: discord.Interaction, message: str, ephemeral: bool = False):
    await interaction.response.send_message(message, ephemeral=ephemeral)
```

### Parameter Types

Slash command parameters are inferred from type annotations:

| Python Type | Discord Type | Notes |
|-------------|-------------|-------|
| `str` | String | Up to 6000 chars |
| `int` | Integer | |
| `float` | Number | |
| `bool` | Boolean | |
| `discord.Member` | User | Resolves to Member in guilds |
| `discord.User` | User | |
| `discord.Role` | Role | |
| `discord.TextChannel` | Channel | Filtered to text channels |
| `discord.VoiceChannel` | Channel | Filtered to voice channels |
| `discord.Attachment` | Attachment | File upload |
| `app_commands.Range[int, 1, 100]` | Integer | With min/max constraints |
| `app_commands.Range[float, 0.0, 1.0]` | Number | With min/max constraints |
| `app_commands.Range[str, 1, 100]` | String | With length constraints |
| `Optional[X]` | X | Makes the parameter optional |

### Choices

```python
@tree.command()
@app_commands.choices(color=[
    app_commands.Choice(name='Red', value='red'),
    app_commands.Choice(name='Blue', value='blue'),
    app_commands.Choice(name='Green', value='green'),
])
async def pick_color(interaction: discord.Interaction, color: app_commands.Choice[str]):
    await interaction.response.send_message(f'You picked: {color.name} ({color.value})')
```

### Autocomplete

```python
async def fruit_autocomplete(interaction: discord.Interaction, current: str):
    fruits = ['Apple', 'Banana', 'Cherry', 'Date']
    return [
        app_commands.Choice(name=f, value=f)
        for f in fruits if current.lower() in f.lower()
    ][:25]  # Max 25 choices

@tree.command()
@app_commands.autocomplete(fruit=fruit_autocomplete)
async def pick_fruit(interaction: discord.Interaction, fruit: str):
    await interaction.response.send_message(f'You picked: {fruit}')
```

### Context Menus (Right-Click Commands)

```python
# User context menu (right-click on user)
@tree.context_menu(name='Get User Info')
async def user_info(interaction: discord.Interaction, user: discord.Member):
    await interaction.response.send_message(f'{user} joined {user.joined_at}', ephemeral=True)

# Message context menu (right-click on message)
@tree.context_menu(name='Report Message')
async def report_message(interaction: discord.Interaction, message: discord.Message):
    await interaction.response.send_message(f'Reported message from {message.author}', ephemeral=True)
```

### Command Groups

```python
# Using Group class
class Settings(app_commands.Group):
    """Manage bot settings"""

    @app_commands.command(description='View a setting')
    async def view(self, interaction: discord.Interaction, key: str):
        await interaction.response.send_message(f'{key} = ...')

    @app_commands.command(description='Set a setting')
    async def set(self, interaction: discord.Interaction, key: str, value: str):
        await interaction.response.send_message(f'Set {key} = {value}')

tree.add_command(Settings(name='settings', description='Bot settings'))
# Creates: /settings view, /settings set
```

### Checks for App Commands

```python
@app_commands.checks.has_permissions(administrator=True)
@app_commands.checks.has_role('Moderator')
@app_commands.checks.cooldown(1, 60.0)                 # 1 use per 60 seconds
@app_commands.checks.bot_has_permissions(send_messages=True)
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)    # Default required perms
```

### Syncing Commands

```python
# Sync globally (takes up to 1 hour to propagate)
await tree.sync()

# Sync to a specific guild (instant)
await tree.sync(guild=discord.Object(id=GUILD_ID))

# Copy global commands to a guild for testing, then sync
tree.copy_global_to(guild=discord.Object(id=GUILD_ID))
await tree.sync(guild=discord.Object(id=GUILD_ID))
```

### Error Handling

```python
@tree.error
async def on_app_command_error(interaction: discord.Interaction, error: app_commands.AppCommandError):
    if isinstance(error, app_commands.MissingPermissions):
        await interaction.response.send_message('You lack permissions.', ephemeral=True)
    elif isinstance(error, app_commands.CommandOnCooldown):
        await interaction.response.send_message(f'Cooldown: {error.retry_after:.1f}s', ephemeral=True)
```

---

## 8. Interactions

The `Interaction` object is passed to all application command and component callbacks.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `int` | Interaction ID |
| `type` | `InteractionType` | Type of interaction |
| `guild_id` | `Optional[int]` | Guild ID |
| `guild` | `Optional[Guild]` | Guild object (cached) |
| `channel` | `Optional[Channel]` | Channel |
| `user` | `Union[User, Member]` | User who triggered |
| `message` | `Optional[Message]` | For component interactions |
| `locale` | `Locale` | User's locale |
| `guild_locale` | `Optional[Locale]` | Guild's locale |
| `entitlements` | `List[Entitlement]` | Monetization entitlements |

### Response Methods

**You must respond within 3 seconds** (or defer first).

```python
# Send a message
await interaction.response.send_message(
    content='Hello!',
    embed=embed,
    embeds=[embed1, embed2],
    view=my_view,
    file=discord.File('image.png'),
    ephemeral=True,                         # Only visible to the user
    tts=False,
    allowed_mentions=discord.AllowedMentions.none(),
)

# Defer — gives you 15 minutes to follow up
await interaction.response.defer(ephemeral=False)
# Then later:
await interaction.followup.send('Done!')

# Edit the message that triggered a component interaction
await interaction.response.edit_message(content='Updated!', view=new_view)

# Send a modal form
await interaction.response.send_modal(my_modal)

# Check if already responded
interaction.response.is_done() -> bool
```

### Follow-up and Editing

```python
# After responding or deferring:
await interaction.followup.send('Another message', ephemeral=True)

# Edit original response
await interaction.edit_original_response(content='Edited!', embed=new_embed)

# Delete original response
await interaction.delete_original_response()
```

---

## 9. UI Components (Views, Buttons, Selects, Modals)

### Views

A `View` is a container that holds interactive components (buttons, selects) on a message.

```python
class MyView(discord.ui.View):
    def __init__(self, timeout: Optional[float] = 180.0):
        super().__init__(timeout=timeout)   # None for no timeout

    @discord.ui.button(label='Confirm', style=discord.ButtonStyle.success)
    async def confirm(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message('Confirmed!', ephemeral=True)
        self.stop()                         # Stop listening

    @discord.ui.button(label='Cancel', style=discord.ButtonStyle.danger)
    async def cancel(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message('Cancelled.', ephemeral=True)
        self.stop()

    async def on_timeout(self):
        # Called when the view times out — disable buttons etc.
        for item in self.children:
            item.disabled = True

# Send with a view
await channel.send('Are you sure?', view=MyView())
```

### Buttons

```python
button = discord.ui.Button(
    style=discord.ButtonStyle.primary,      # primary, secondary, success, danger, link
    label='Click Me',
    emoji='👍',                              # Optional emoji
    custom_id='my_button',                  # Unique ID (auto-generated if using decorators)
    url='https://example.com',              # Only for ButtonStyle.link (no callback)
    disabled=False,
    row=0,                                  # Row 0-4 (5 rows max, 5 items per row)
)
```

| Style | Color | Has Callback |
|-------|-------|-------------|
| `ButtonStyle.primary` / `blurple` | Blurple | Yes |
| `ButtonStyle.secondary` / `grey` | Gray | Yes |
| `ButtonStyle.success` / `green` | Green | Yes |
| `ButtonStyle.danger` / `red` | Red | Yes |
| `ButtonStyle.link` | Gray | No (opens URL) |

### Select Menus

```python
class MyView(discord.ui.View):
    @discord.ui.select(
        placeholder='Choose an option...',
        min_values=1,
        max_values=1,
        options=[
            discord.SelectOption(label='Option 1', value='opt1', description='First option', emoji='1️⃣'),
            discord.SelectOption(label='Option 2', value='opt2', description='Second option', emoji='2️⃣'),
            discord.SelectOption(label='Option 3', value='opt3', default=True),  # Pre-selected
        ]
    )
    async def select_callback(self, interaction: discord.Interaction, select: discord.ui.Select):
        await interaction.response.send_message(f'You chose: {select.values[0]}')
```

### Typed Select Menus

```python
# Select users
@discord.ui.select(cls=discord.ui.UserSelect, placeholder='Pick a user')
async def user_select(self, interaction, select: discord.ui.UserSelect):
    user = select.values[0]  # discord.Member or discord.User

# Select roles
@discord.ui.select(cls=discord.ui.RoleSelect, placeholder='Pick a role')
async def role_select(self, interaction, select: discord.ui.RoleSelect):
    role = select.values[0]  # discord.Role

# Select channels
@discord.ui.select(cls=discord.ui.ChannelSelect, placeholder='Pick a channel',
                    channel_types=[discord.ChannelType.text])
async def channel_select(self, interaction, select: discord.ui.ChannelSelect):
    channel = select.values[0]  # discord.AppCommandChannel

# Select users or roles
@discord.ui.select(cls=discord.ui.MentionableSelect)
async def mention_select(self, interaction, select: discord.ui.MentionableSelect):
    pass
```

### Modals (Form Dialogs)

```python
class FeedbackModal(discord.ui.Modal, title='Send Feedback'):
    name = discord.ui.TextInput(
        label='Your Name',
        placeholder='Enter your name...',
        style=discord.TextStyle.short,      # Single line
        min_length=1,
        max_length=100,
        required=True,
    )
    feedback = discord.ui.TextInput(
        label='Feedback',
        placeholder='Tell us what you think...',
        style=discord.TextStyle.paragraph,  # Multi-line
        max_length=4000,
        required=True,
    )

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.send_message(
            f'Thanks {self.name.value}! Feedback received.',
            ephemeral=True
        )

    async def on_error(self, interaction: discord.Interaction, error: Exception):
        await interaction.response.send_message('Something went wrong.', ephemeral=True)

# Send a modal (must be from an interaction response)
await interaction.response.send_modal(FeedbackModal())
```

### Persistent Views

Views that survive bot restarts by using fixed `custom_id` values.

```python
class PersistentView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)      # No timeout

    @discord.ui.button(label='Ticket', style=discord.ButtonStyle.primary, custom_id='persistent:ticket')
    async def ticket_button(self, interaction, button):
        await interaction.response.send_message('Ticket created!', ephemeral=True)

# Register on startup
@bot.event
async def setup_hook():
    bot.add_view(PersistentView())          # Must be added before bot receives interactions
```

---

## 10. Messages

### Sending Messages

```python
# Via any Messageable (TextChannel, DMChannel, Thread, Context, Member, User)
message = await channel.send(
    content='Hello!',                       # Text content
    embed=embed,                            # Single embed
    embeds=[embed1, embed2],                # Multiple embeds (max 10)
    file=discord.File('image.png'),         # Single file
    files=[file1, file2],                   # Multiple files (max 10)
    view=my_view,                           # UI components
    tts=False,                              # Text-to-speech
    delete_after=10.0,                      # Auto-delete after seconds
    allowed_mentions=discord.AllowedMentions.none(),
    reference=message_to_reply_to,          # Reply to a message
    mention_author=False,                   # Ping the replied-to author
    suppress_embeds=False,                  # Suppress link embeds
    silent=False,                           # No notification
    stickers=[sticker],                     # Stickers
)
```

### Message Object

```python
class Message:
    id: int
    channel: Union[TextChannel, DMChannel, Thread, ...]
    guild: Optional[Guild]
    author: Union[User, Member]
    content: str                            # Requires message_content intent
    timestamp: datetime
    edited_at: Optional[datetime]
    tts: bool
    mention_everyone: bool
    mentions: List[Union[User, Member]]
    role_mentions: List[Role]
    channel_mentions: List[GuildChannel]
    attachments: List[Attachment]
    embeds: List[Embed]
    reactions: List[Reaction]
    pinned: bool
    type: MessageType
    reference: Optional[MessageReference]   # If reply
    jump_url: str                           # URL to jump to message

    # Methods
    await message.edit(content='Updated', embed=new_embed)
    await message.delete(delay=None)
    await message.pin(reason=None)
    await message.unpin(reason=None)
    await message.reply(content='Reply text')
    await message.add_reaction('👍')
    await message.add_reaction(emoji_object)
    await message.remove_reaction('👍', member)
    await message.clear_reactions()
    await message.clear_reaction('👍')
    await message.create_thread(name='Discussion', auto_archive_duration=60)
```

### Attachment

```python
class Attachment:
    id: int
    filename: str
    size: int                               # Bytes
    url: str
    proxy_url: str
    content_type: Optional[str]             # MIME type
    width: Optional[int]                    # For images
    height: Optional[int]                   # For images
    is_spoiler() -> bool
    is_voice_message() -> bool

    await attachment.save(fp='downloaded.png')  # Save to file
    await attachment.read() -> bytes            # Read bytes
    await attachment.to_file() -> File          # Convert to File for re-sending
```

---

## 11. Embeds

```python
embed = discord.Embed(
    title='My Embed',
    description='A description here',
    url='https://example.com',              # Title becomes clickable link
    colour=discord.Colour.blue(),           # Or color= (alias)
    timestamp=datetime.datetime.now(datetime.timezone.utc),
)
embed.set_author(name='Author Name', url='https://...', icon_url='https://...')
embed.set_footer(text='Footer text', icon_url='https://...')
embed.set_image(url='https://...')          # Large image below fields
embed.set_thumbnail(url='https://...')      # Small image top-right
embed.add_field(name='Field 1', value='Value 1', inline=True)
embed.add_field(name='Field 2', value='Value 2', inline=True)
embed.add_field(name='Field 3', value='Value 3', inline=False)

# Modify
embed.insert_field_at(0, name='First', value='Inserted')
embed.set_field_at(0, name='Updated', value='Changed')
embed.remove_field(0)
embed.clear_fields()
embed.remove_author()

# Limits
# Title: 256 chars, Description: 4096 chars, Fields: 25 max
# Field name: 256 chars, Field value: 1024 chars
# Footer text: 2048 chars, Author name: 256 chars
# Total characters across all embed elements: 6000
```

---

## 12. Guilds (Servers)

### Key Properties

```python
class Guild:
    id: int
    name: str
    description: Optional[str]
    owner_id: int
    owner: Optional[Member]
    icon: Optional[Asset]
    banner: Optional[Asset]
    splash: Optional[Asset]
    member_count: int
    max_members: Optional[int]
    verification_level: VerificationLevel
    default_notifications: NotificationLevel
    explicit_content_filter: ContentFilter
    mfa_level: MFALevel
    features: Sequence[str]
    premium_tier: int                       # Boost level (0-3)
    premium_subscription_count: int         # Boost count
    created_at: datetime

    # Cached collections
    roles: Sequence[Role]
    members: Sequence[Member]
    channels: Sequence[GuildChannel]
    threads: Sequence[Thread]
    text_channels: List[TextChannel]
    voice_channels: List[VoiceChannel]
    stage_channels: List[StageChannel]
    categories: List[CategoryChannel]
    forums: List[ForumChannel]
    emojis: Sequence[Emoji]
    stickers: Sequence[GuildSticker]

    # Special channels
    system_channel: Optional[TextChannel]
    rules_channel: Optional[TextChannel]
    afk_channel: Optional[VoiceChannel]
```

### Key Methods

```python
# Fetch from API
await guild.fetch_member(user_id) -> Member
await guild.fetch_members(limit=1000) -> AsyncIterator[Member]
guild.fetch_bans(limit=1000) -> AsyncIterator[BanEntry]

# Cache lookups
guild.get_channel(id) -> Optional[GuildChannel]
guild.get_role(id) -> Optional[Role]
guild.get_member(user_id) -> Optional[Member]
guild.get_member_named('Username') -> Optional[Member]

# Creation
await guild.create_text_channel('general', category=category, topic='Welcome')
await guild.create_voice_channel('Voice Chat', category=category)
await guild.create_category('My Category')
await guild.create_role(name='Moderator', color=discord.Colour.blue(),
                         permissions=discord.Permissions(kick_members=True))
await guild.create_custom_emoji(name='my_emoji', image=emoji_bytes)

# Moderation
await guild.kick(member, reason='Reason')
await guild.ban(user, reason='Reason', delete_message_days=1)
await guild.unban(user, reason='Reason')
await guild.prune_members(days=30, compute_prune_count=True)

# Edit
await guild.edit(name='New Name', icon=icon_bytes, banner=banner_bytes)
await guild.leave()
```

---

## 13. Channels

### Channel Type Hierarchy

```
abc.GuildChannel (base for all guild channels)
├── TextChannel (abc.Messageable)
├── VoiceChannel (abc.Messageable, abc.Connectable)
├── StageChannel (abc.Connectable)
├── CategoryChannel
├── ForumChannel
└── MediaChannel

abc.PrivateChannel
├── DMChannel (abc.Messageable)
└── GroupChannel (abc.Messageable)
```

### TextChannel

```python
class TextChannel:
    id: int
    name: str
    guild: Guild
    category: Optional[CategoryChannel]
    topic: Optional[str]
    position: int
    slowmode_delay: int                     # Seconds (0 = disabled)
    nsfw: bool
    default_auto_archive_duration: int
    last_message_id: Optional[int]

    await channel.send(content='Hello')
    await channel.fetch_message(message_id) -> Message
    channel.history(limit=100, oldest_first=False) -> AsyncIterator[Message]
    await channel.pins() -> List[Message]
    await channel.purge(limit=100, check=lambda m: m.author.bot)
    await channel.create_thread(name='Thread', message=msg, auto_archive_duration=60)
    await channel.create_invite(max_age=3600, max_uses=10)
    await channel.edit(name='new-name', topic='New topic', slowmode_delay=5)
    await channel.delete(reason='Cleanup')
    channel.permissions_for(member) -> Permissions
```

### VoiceChannel

```python
class VoiceChannel:
    bitrate: int
    user_limit: int
    rtc_region: Optional[str]
    members: List[Member]                   # Currently connected members

    await channel.connect() -> VoiceClient
    await channel.edit(name='New Name', bitrate=96000, user_limit=10)
```

### ForumChannel

```python
class ForumChannel:
    topic: Optional[str]
    default_auto_archive_duration: int
    default_layout: ForumLayoutType
    default_sort_order: Optional[ForumOrderType]
    available_tags: List[ForumTag]

    # Create a forum post (returns Thread + initial Message)
    thread, message = await forum.create_thread(
        name='My Post',
        content='Post body',
        embed=embed,
        file=file,
        applied_tags=[tag],
        auto_archive_duration=1440,
    )
```

### CategoryChannel

```python
class CategoryChannel:
    channels: List[GuildChannel]            # Channels in this category
    text_channels: List[TextChannel]
    voice_channels: List[VoiceChannel]

    await category.create_text_channel('general')
    await category.create_voice_channel('Voice Chat')
```

---

## 14. Members and Users

### User

```python
class User:
    id: int
    name: str                               # Username
    discriminator: str                      # '0' for new usernames
    global_name: Optional[str]              # Display name
    avatar: Optional[Asset]
    bot: bool
    system: bool
    created_at: datetime
    display_name: str                       # global_name or name
    mention: str                            # '<@id>'

    await user.send('Hello via DM')         # Send DM
    await user.create_dm() -> DMChannel
```

### Member (User in a Guild)

```python
class Member(User):                         # Inherits all User properties
    guild: Guild
    nick: Optional[str]                     # Server nickname
    roles: List[Role]                       # Ordered by position
    joined_at: Optional[datetime]
    premium_since: Optional[datetime]       # Boosting since
    display_name: str                       # nick > global_name > name
    top_role: Role                          # Highest role
    guild_permissions: Permissions
    voice: Optional[VoiceState]             # None if not in voice
    timed_out_until: Optional[datetime]

    # Moderation
    await member.kick(reason='Reason')
    await member.ban(reason='Reason', delete_message_days=1)
    await member.edit(nick='New Nick', roles=[role1, role2],
                       mute=False, deafen=False, timed_out_until=timeout_dt)
    await member.add_roles(role1, role2, reason='Added roles')
    await member.remove_roles(role1, reason='Removed role')
    await member.move_to(voice_channel)     # Move in voice
    await member.timeout(duration, reason='Timeout reason')
```

---

## 15. Roles

```python
class Role:
    id: int
    guild: Guild
    name: str
    colour: Colour                          # color is an alias
    hoist: bool                             # Displayed separately in member list
    position: int
    permissions: Permissions
    managed: bool                           # Managed by integration/bot
    mentionable: bool
    icon: Optional[Asset]
    emoji: Optional[PartialEmoji]
    created_at: datetime
    mention: str                            # '<@&id>'
    members: List[Member]                   # Members with this role

    await role.edit(name='New Name', colour=discord.Colour.red(),
                     permissions=discord.Permissions(send_messages=True),
                     hoist=True, mentionable=True, reason='Updated')
    await role.delete(reason='Cleanup')

    # Comparison operators work by position
    role1 > role2   # role1 is higher
    role1 < role2   # role1 is lower
```

---

## 16. Permissions

```python
# Create with keyword arguments
perms = discord.Permissions(
    send_messages=True,
    manage_messages=True,
    administrator=False,
)

# Presets
discord.Permissions.all()
discord.Permissions.none()
discord.Permissions.general()
discord.Permissions.text()
discord.Permissions.voice()

# Check permissions
if member.guild_permissions.administrator:
    ...
if channel.permissions_for(member).send_messages:
    ...

# Update
perms.update(send_messages=False)
```

### Available Permission Flags

`create_instant_invite`, `kick_members`, `ban_members`, `administrator`, `manage_channels`,
`manage_guild`, `add_reactions`, `view_audit_log`, `priority_speaker`, `stream`,
`view_channel`, `send_messages`, `send_tts_messages`, `manage_messages`, `embed_links`,
`attach_files`, `read_message_history`, `mention_everyone`, `use_external_emojis`,
`view_guild_insights`, `connect`, `speak`, `mute_members`, `deafen_members`, `move_members`,
`use_voice_activation`, `change_nickname`, `manage_nicknames`, `manage_roles`,
`manage_webhooks`, `manage_expressions`, `use_application_commands`, `request_to_speak`,
`manage_events`, `manage_threads`, `create_public_threads`, `create_private_threads`,
`use_external_stickers`, `send_messages_in_threads`, `use_embedded_activities`,
`moderate_members`, `send_polls`, `create_events`

### Permission Overwrites (Channel-Level)

```python
overwrites = {
    guild.default_role: discord.PermissionOverwrite(read_messages=False),
    guild.me: discord.PermissionOverwrite(read_messages=True),
    some_role: discord.PermissionOverwrite(send_messages=True, read_messages=True),
}
await guild.create_text_channel('private', overwrites=overwrites)
```

---

## 17. Threads

```python
class Thread:
    id: int
    name: str
    guild: Guild
    parent: Optional[Union[TextChannel, ForumChannel]]
    owner_id: int
    owner: Optional[Member]
    created_at: datetime
    archive_timestamp: datetime
    auto_archive_duration: int              # Minutes: 60, 1440, 4320, 10080
    archived: bool
    locked: bool
    message_count: Optional[int]
    member_count: Optional[int]

    await thread.send(content='Hello thread!')
    await thread.join()
    await thread.leave()
    await thread.add_user(user)
    await thread.remove_user(user)
    await thread.edit(name='New Name', archived=True, locked=True, auto_archive_duration=1440)
    await thread.delete()
    thread.history(limit=100) -> AsyncIterator[Message]
```

---

## 18. Webhooks

```python
# Create a webhook in a channel
webhook = await channel.create_webhook(name='My Webhook', avatar=avatar_bytes)

# Get from URL
webhook = discord.Webhook.from_url(
    'https://discord.com/api/webhooks/ID/TOKEN',
    session=aiohttp.ClientSession()         # Required for async version
)

# Send
await webhook.send(
    content='Hello from webhook!',
    username='Custom Name',                 # Override display name
    avatar_url='https://...',               # Override avatar
    embed=embed,
    file=file,
    thread=thread_object,                   # Send to a thread
    wait=True,                              # Return the created Message
)

# Edit/Delete
await webhook.edit(name='New Name', avatar=new_bytes)
await webhook.delete()

# Sync version (no async, no aiohttp needed — uses requests)
sync_webhook = discord.SyncWebhook.from_url(url)
sync_webhook.send('Hello!')
```

---

## 19. Voice

### Connecting

```python
# Connect to a voice channel
voice_client = await voice_channel.connect()

# Or from a guild member's voice state
if ctx.author.voice:
    voice_client = await ctx.author.voice.channel.connect()
```

### Playing Audio

```python
import discord

# From file
source = discord.FFmpegPCMAudio('song.mp3')
voice_client.play(source, after=lambda e: print(f'Finished: {e}'))

# From URL
source = discord.FFmpegPCMAudio('https://example.com/audio.mp3')

# With volume control
source = discord.PCMVolumeTransformer(discord.FFmpegPCMAudio('song.mp3'), volume=0.5)
voice_client.play(source)
source.volume = 0.75                        # Adjust volume (0.0 to 2.0)

# Control playback
voice_client.pause()
voice_client.resume()
voice_client.stop()
voice_client.is_playing() -> bool
voice_client.is_paused() -> bool

# Disconnect
await voice_client.disconnect()

# Move to another channel
await voice_client.move_to(other_voice_channel)
```

> **Note:** `FFmpegPCMAudio` requires FFmpeg to be installed and on PATH.

---

## 20. Enums

### Key Enums

```python
# Channel types
discord.ChannelType.text            # 0
discord.ChannelType.voice           # 2
discord.ChannelType.private         # 1 (DM)
discord.ChannelType.category        # 4
discord.ChannelType.news            # 5
discord.ChannelType.stage_voice     # 13
discord.ChannelType.forum           # 15
discord.ChannelType.public_thread   # 11
discord.ChannelType.private_thread  # 12

# Status
discord.Status.online
discord.Status.offline
discord.Status.idle
discord.Status.dnd                  # Do not disturb
discord.Status.invisible

# Activity type
discord.ActivityType.playing        # "Playing ..."
discord.ActivityType.streaming      # "Streaming ..."
discord.ActivityType.listening      # "Listening to ..."
discord.ActivityType.watching       # "Watching ..."
discord.ActivityType.competing      # "Competing in ..."
discord.ActivityType.custom         # Custom status

# Button style
discord.ButtonStyle.primary         # Blurple
discord.ButtonStyle.secondary       # Gray
discord.ButtonStyle.success         # Green
discord.ButtonStyle.danger          # Red
discord.ButtonStyle.link            # Gray, opens URL

# Text input style
discord.TextStyle.short             # Single line
discord.TextStyle.paragraph         # Multi-line

# Interaction type
discord.InteractionType.application_command
discord.InteractionType.message_component
discord.InteractionType.application_command_autocomplete
discord.InteractionType.modal_submit

# Verification level
discord.VerificationLevel.none
discord.VerificationLevel.low       # Verified email
discord.VerificationLevel.medium    # 5-minute account age
discord.VerificationLevel.high      # 10-minute guild member
discord.VerificationLevel.highest   # Verified phone

# Message type
discord.MessageType.default
discord.MessageType.pins_add
discord.MessageType.member_join
discord.MessageType.reply
discord.MessageType.chat_input_command
discord.MessageType.context_menu_command
discord.MessageType.thread_starter_message
```

---

## 21. Errors and Exceptions

### Exception Hierarchy

```
DiscordException (base)
├── ClientException
│   ├── InvalidData
│   └── LoginFailure
├── GatewayNotFound
├── ConnectionClosed
├── PrivilegedIntentsRequired
├── InteractionResponded
├── HTTPException
│   ├── Forbidden (403)
│   ├── NotFound (404)
│   └── DiscordServerError (500+)
├── RateLimited
└── opus.OpusNotLoaded

ext.commands errors:
├── CommandError
│   ├── CommandNotFound
│   ├── MissingRequiredArgument
│   ├── BadArgument
│   ├── CheckFailure
│   │   ├── MissingPermissions
│   │   ├── BotMissingPermissions
│   │   ├── MissingRole
│   │   ├── MissingAnyRole
│   │   ├── NotOwner
│   │   ├── NSFWChannelRequired
│   │   └── PrivateMessageOnly / NoPrivateMessage
│   ├── CommandOnCooldown
│   ├── MaxConcurrencyReached
│   ├── DisabledCommand
│   └── CommandInvokeError

app_commands errors:
├── AppCommandError
│   ├── CommandNotFound
│   ├── MissingPermissions
│   ├── BotMissingPermissions
│   ├── CommandOnCooldown
│   ├── CheckFailure
│   ├── TransformerError
│   └── CommandSignatureMismatch
```

### Common Error Handling

```python
# Prefix command errors
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send(f'Missing permissions: {error.missing_permissions}')
    elif isinstance(error, commands.CommandOnCooldown):
        await ctx.send(f'Try again in {error.retry_after:.1f}s')
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f'Missing argument: {error.param.name}')
    elif isinstance(error, commands.CommandNotFound):
        pass  # Silently ignore unknown commands

# HTTPException details
try:
    await channel.send('...')
except discord.HTTPException as e:
    print(e.status)     # HTTP status code
    print(e.code)       # Discord error code
    print(e.text)       # Error message
```

---

## 22. Events

Register event handlers with `@bot.event` or `@bot.listen()` (for multiple handlers).

### Connection Events

```python
@bot.event
async def on_ready():
    """Called when the bot has connected and cached all guilds."""
    print(f'Logged in as {bot.user}')

@bot.event
async def on_connect():
    """Called when the bot successfully connects to the gateway."""

@bot.event
async def on_disconnect():
    """Called when the bot disconnects from the gateway."""

@bot.event
async def on_resumed():
    """Called when the bot resumes a session."""
```

### Message Events

```python
@bot.event
async def on_message(message: discord.Message):
    """Called on every message the bot can see."""
    if message.author == bot.user:
        return
    # IMPORTANT: process commands if overriding on_message in a Bot
    await bot.process_commands(message)

@bot.event
async def on_message_edit(before: discord.Message, after: discord.Message):
    ...

@bot.event
async def on_message_delete(message: discord.Message):
    ...

@bot.event
async def on_bulk_message_delete(messages: list[discord.Message]):
    ...

@bot.event
async def on_reaction_add(reaction: discord.Reaction, user: Union[discord.Member, discord.User]):
    ...

@bot.event
async def on_reaction_remove(reaction: discord.Reaction, user: Union[discord.Member, discord.User]):
    ...
```

### Member Events

```python
@bot.event
async def on_member_join(member: discord.Member):
    ...

@bot.event
async def on_member_remove(member: discord.Member):
    ...

@bot.event
async def on_member_update(before: discord.Member, after: discord.Member):
    """Nickname, role, avatar, or timeout changes."""

@bot.event
async def on_user_update(before: discord.User, after: discord.User):
    """Username, avatar, or discriminator changes."""

@bot.event
async def on_presence_update(before: discord.Member, after: discord.Member):
    """Status or activity changes. Requires presences intent."""
```

### Guild Events

```python
@bot.event
async def on_guild_join(guild: discord.Guild):
    """Bot was added to a guild."""

@bot.event
async def on_guild_remove(guild: discord.Guild):
    """Bot was removed from a guild."""

@bot.event
async def on_guild_update(before: discord.Guild, after: discord.Guild):
    ...

@bot.event
async def on_guild_available(guild: discord.Guild):
    ...

@bot.event
async def on_guild_unavailable(guild: discord.Guild):
    ...
```

### Channel Events

```python
@bot.event
async def on_guild_channel_create(channel: discord.abc.GuildChannel):
    ...

@bot.event
async def on_guild_channel_delete(channel: discord.abc.GuildChannel):
    ...

@bot.event
async def on_guild_channel_update(before, after):
    ...
```

### Voice Events

```python
@bot.event
async def on_voice_state_update(member: discord.Member, before: discord.VoiceState, after: discord.VoiceState):
    """Member joins, leaves, or moves voice channels."""
    if before.channel is None and after.channel is not None:
        print(f'{member} joined {after.channel}')
    elif before.channel is not None and after.channel is None:
        print(f'{member} left {before.channel}')
```

### Role Events

```python
@bot.event
async def on_guild_role_create(role: discord.Role):
    ...

@bot.event
async def on_guild_role_delete(role: discord.Role):
    ...

@bot.event
async def on_guild_role_update(before: discord.Role, after: discord.Role):
    ...
```

### Thread Events

```python
@bot.event
async def on_thread_create(thread: discord.Thread):
    ...

@bot.event
async def on_thread_join(thread: discord.Thread):
    ...

@bot.event
async def on_thread_remove(thread: discord.Thread):
    ...

@bot.event
async def on_thread_delete(thread: discord.Thread):
    ...

@bot.event
async def on_thread_member_join(member: discord.ThreadMember):
    ...

@bot.event
async def on_thread_member_remove(member: discord.ThreadMember):
    ...
```

### Invite Events

```python
@bot.event
async def on_invite_create(invite: discord.Invite):
    ...

@bot.event
async def on_invite_delete(invite: discord.Invite):
    ...
```

### Interaction Events

```python
@bot.event
async def on_interaction(interaction: discord.Interaction):
    """Called on any interaction (commands, components, autocomplete, modals)."""
```

### Scheduled Event Events

```python
@bot.event
async def on_scheduled_event_create(event):
    ...

@bot.event
async def on_scheduled_event_update(before, after):
    ...

@bot.event
async def on_scheduled_event_delete(event):
    ...

@bot.event
async def on_scheduled_event_user_add(event, user):
    ...

@bot.event
async def on_scheduled_event_user_remove(event, user):
    ...
```

### AutoMod Events

```python
@bot.event
async def on_automod_rule_create(rule):
    ...

@bot.event
async def on_automod_rule_update(rule):
    ...

@bot.event
async def on_automod_rule_delete(rule):
    ...

@bot.event
async def on_automod_action(execution):
    ...
```

### Raw Events

Raw events fire even when the object isn't in cache:

```python
@bot.event
async def on_raw_message_delete(payload: discord.RawMessageDeleteEvent):
    print(payload.message_id, payload.channel_id, payload.guild_id)

@bot.event
async def on_raw_message_edit(payload: discord.RawMessageUpdateEvent):
    ...

@bot.event
async def on_raw_reaction_add(payload: discord.RawReactionActionEvent):
    print(payload.emoji, payload.user_id, payload.message_id)

@bot.event
async def on_raw_reaction_remove(payload: discord.RawReactionActionEvent):
    ...

@bot.event
async def on_raw_bulk_message_delete(payload: discord.RawBulkMessageDeleteEvent):
    ...
```

---

## 23. Tasks (ext.tasks)

Background task loops with built-in reconnection and error handling.

```python
from discord.ext import tasks

class MyCog(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.my_task.start()                # Start the loop

    def cog_unload(self):
        self.my_task.cancel()               # Stop on cog unload

    @tasks.loop(seconds=60.0)               # Run every 60 seconds
    async def my_task(self):
        channel = self.bot.get_channel(CHANNEL_ID)
        await channel.send('Periodic message!')

    @my_task.before_loop
    async def before_my_task(self):
        await self.bot.wait_until_ready()   # Wait for bot to be ready

    @my_task.after_loop
    async def after_my_task(self):
        print('Task finished')

    @my_task.error
    async def my_task_error(self, error: Exception):
        print(f'Task error: {error}')
```

### Loop Parameters

```python
@tasks.loop(
    seconds=0.0,                            # Interval in seconds
    minutes=0.0,                            # Interval in minutes
    hours=0.0,                              # Interval in hours
    time=datetime.time(hour=12, minute=0),  # Run at specific time(s) daily
    count=None,                             # Number of iterations (None = infinite)
    reconnect=True,                         # Retry on connection errors
    name=None,                              # Name for debugging
)
```

### Loop Methods

```python
loop_task.start(*args, **kwargs)            # Start the loop
loop_task.stop()                            # Stop after current iteration
loop_task.cancel()                          # Cancel immediately
loop_task.restart(*args, **kwargs)          # Restart the loop
loop_task.is_running() -> bool
loop_task.is_being_cancelled() -> bool
loop_task.failed() -> bool
loop_task.current_loop -> int               # Current iteration count
loop_task.next_iteration -> Optional[datetime]
loop_task.change_interval(seconds=30)       # Change interval
```

---

## 24. Converters (ext.commands)

Converters transform string arguments from prefix commands into Python objects.

### Built-in Converters (via type annotations)

```python
@bot.command()
async def info(ctx,
    member: discord.Member,                 # Resolves mention, ID, name#discrim, name
    channel: discord.TextChannel,           # Resolves mention, ID, name
    role: discord.Role,                     # Resolves mention, ID, name
    emoji: discord.Emoji,                   # Resolves emoji string or ID
    colour: discord.Colour,                 # Resolves hex (#ff0000), 0xff0000, rgb(r,g,b)
    message: discord.Message,               # Resolves channel_id-message_id or jump URL
    invite: discord.Invite,                 # Resolves invite URL/code
    guild: discord.Guild,                   # Resolves guild ID
    user: discord.User,                     # Resolves ID, mention (fetches from API if needed)
):
    pass
```

### Special Converters

```python
# Greedy — consumes arguments until conversion fails
@bot.command()
async def ban(ctx, members: commands.Greedy[discord.Member], *, reason: str):
    for member in members:
        await member.ban(reason=reason)

# Literal — exact string match
from typing import Literal
@bot.command()
async def mode(ctx, mode: Literal['easy', 'hard']):
    pass

# Optional with default
from typing import Optional
@bot.command()
async def info(ctx, member: Optional[discord.Member] = None):
    member = member or ctx.author
```

### Custom Converter

```python
class DurationConverter(commands.Converter):
    async def convert(self, ctx, argument: str) -> int:
        """Convert '10s', '5m', '2h' to seconds."""
        units = {'s': 1, 'm': 60, 'h': 3600}
        try:
            return int(argument[:-1]) * units[argument[-1]]
        except (KeyError, ValueError):
            raise commands.BadArgument(f'Invalid duration: {argument}')

@bot.command()
async def timeout(ctx, member: discord.Member, duration: DurationConverter):
    ...
```

---

## 25. Utility Classes (File, Colour, AllowedMentions)

### File

```python
# From file path
file = discord.File('image.png')
file = discord.File('secret.txt', filename='revealed.txt')  # Rename
file = discord.File('image.png', spoiler=True)               # Mark as spoiler
file = discord.File('image.png', description='Alt text')     # For accessibility

# From bytes
import io
buffer = io.BytesIO(image_bytes)
file = discord.File(buffer, filename='image.png')

# Send
await channel.send(file=file)
await channel.send(files=[file1, file2])    # Max 10 files
```

### Colour (Color)

```python
# From hex
colour = discord.Colour(0x3498db)
colour = discord.Colour.from_rgb(52, 152, 219)

# Presets
discord.Colour.red()
discord.Colour.green()
discord.Colour.blue()
discord.Colour.yellow()
discord.Colour.orange()
discord.Colour.purple()
discord.Colour.gold()
discord.Colour.teal()
discord.Colour.dark_blue()
discord.Colour.dark_green()
discord.Colour.dark_red()
discord.Colour.blurple()                    # Discord brand color
discord.Colour.greyple()
discord.Colour.dark_theme()                 # Dark mode background
discord.Colour.random()                     # Random color
discord.Colour.default()                    # No color (invisible sidebar)

# Properties
colour.r                                    # Red component (0-255)
colour.g                                    # Green component (0-255)
colour.b                                    # Blue component (0-255)
colour.value                                # Integer value
```

### AllowedMentions

```python
# Default: mentions everyone=False, users=True, roles=True, replied_user=True
mentions = discord.AllowedMentions(
    everyone=False,                         # @everyone and @here
    users=True,                             # User mentions
    roles=False,                            # Role mentions
    replied_user=False,                     # Ping on reply
)

# No mentions at all
discord.AllowedMentions.none()

# All mentions
discord.AllowedMentions.all()

# Set as default for the bot
bot = commands.Bot(..., allowed_mentions=discord.AllowedMentions(everyone=False))

# Override per message
await channel.send('@everyone', allowed_mentions=discord.AllowedMentions.none())
```

---

## 26. Common Patterns

### Minimal Bot

```python
import discord
from discord.ext import commands

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix='!', intents=intents)

@bot.event
async def on_ready():
    print(f'{bot.user} is online!')

@bot.command()
async def ping(ctx):
    await ctx.send(f'Pong! {round(bot.latency * 1000)}ms')

bot.run('YOUR_TOKEN')
```

### Slash Commands Only (No Prefix Commands)

```python
import discord

intents = discord.Intents.default()
client = discord.Client(intents=intents)
tree = discord.app_commands.CommandTree(client)

@tree.command(name='hello', description='Say hello')
async def hello(interaction: discord.Interaction):
    await interaction.response.send_message(f'Hello {interaction.user.mention}!')

@client.event
async def on_ready():
    await tree.sync()
    print(f'{client.user} is online!')

client.run('YOUR_TOKEN')
```

### Hybrid Commands (Prefix + Slash)

```python
from discord.ext import commands

bot = commands.Bot(command_prefix='!', intents=intents)

@bot.hybrid_command(name='ping', description='Check latency')
async def ping(ctx: commands.Context):
    """Works as both !ping and /ping"""
    await ctx.send(f'Pong! {round(bot.latency * 1000)}ms')

@bot.event
async def setup_hook():
    await bot.tree.sync()
```

### Confirmation Prompt with Buttons

```python
class ConfirmView(discord.ui.View):
    def __init__(self, author_id: int):
        super().__init__(timeout=30)
        self.value = None
        self.author_id = author_id

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        return interaction.user.id == self.author_id

    @discord.ui.button(label='Confirm', style=discord.ButtonStyle.success)
    async def confirm(self, interaction, button):
        self.value = True
        self.stop()
        await interaction.response.edit_message(content='Confirmed!', view=None)

    @discord.ui.button(label='Cancel', style=discord.ButtonStyle.danger)
    async def cancel(self, interaction, button):
        self.value = False
        self.stop()
        await interaction.response.edit_message(content='Cancelled.', view=None)

# Usage
view = ConfirmView(ctx.author.id)
await ctx.send('Are you sure?', view=view)
await view.wait()
if view.value:
    # Proceed
    ...
```

### Paginated Embeds

```python
class PaginatorView(discord.ui.View):
    def __init__(self, pages: list[discord.Embed]):
        super().__init__(timeout=120)
        self.pages = pages
        self.current = 0

    @discord.ui.button(emoji='◀', style=discord.ButtonStyle.secondary)
    async def prev_page(self, interaction, button):
        self.current = max(0, self.current - 1)
        await interaction.response.edit_message(embed=self.pages[self.current])

    @discord.ui.button(emoji='▶', style=discord.ButtonStyle.secondary)
    async def next_page(self, interaction, button):
        self.current = min(len(self.pages) - 1, self.current + 1)
        await interaction.response.edit_message(embed=self.pages[self.current])
```

### Cog with Extension Loading

```python
# cogs/moderation.py
import discord
from discord.ext import commands

class Moderation(commands.Cog):
    def __init__(self, bot):
        self.bot = bot

    @commands.command()
    @commands.has_permissions(ban_members=True)
    async def ban(self, ctx, member: discord.Member, *, reason=None):
        await member.ban(reason=reason)
        await ctx.send(f'Banned {member}')

async def setup(bot):
    await bot.add_cog(Moderation(bot))

# main.py
import asyncio
import discord
from discord.ext import commands

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix='!', intents=intents)

async def main():
    async with bot:
        await bot.load_extension('cogs.moderation')
        await bot.start('YOUR_TOKEN')

asyncio.run(main())
```

### Error Handler with Logging

```python
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        return
    elif isinstance(error, commands.MissingPermissions):
        await ctx.send(f'You need: {", ".join(error.missing_permissions)}')
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f'Missing argument: `{error.param.name}`\nUsage: `{ctx.prefix}{ctx.command.qualified_name} {ctx.command.signature}`')
    elif isinstance(error, commands.CommandOnCooldown):
        await ctx.send(f'Cooldown: try again in {error.retry_after:.1f}s')
    elif isinstance(error, commands.BotMissingPermissions):
        await ctx.send(f'I need: {", ".join(error.missing_permissions)}')
    else:
        raise error  # Re-raise unexpected errors
```

### Wait For Pattern

```python
@bot.command()
async def ask(ctx):
    await ctx.send('What is your name?')
    try:
        msg = await bot.wait_for(
            'message',
            check=lambda m: m.author == ctx.author and m.channel == ctx.channel,
            timeout=30.0
        )
    except asyncio.TimeoutError:
        await ctx.send('You took too long!')
    else:
        await ctx.send(f'Hello, {msg.content}!')
```

### Using discord.utils

```python
# Find first match in an iterable
role = discord.utils.get(guild.roles, name='Moderator')
channel = discord.utils.get(guild.text_channels, name='general')
member = discord.utils.find(lambda m: m.nick == 'Test', guild.members)

# Snowflake timestamp
created_at = discord.utils.snowflake_time(message_id)

# Current UTC time
now = discord.utils.utcnow()

# Format datetime for Discord display
discord.utils.format_dt(dt, style='F')     # Full date/time
discord.utils.format_dt(dt, style='R')     # Relative ("2 hours ago")
# Styles: 't' short time, 'T' long time, 'd' short date, 'D' long date,
#          'f' short datetime, 'F' long datetime, 'R' relative
```

---

## Quick Reference: Key Gotchas

1. **`intents` is required** — `Client(intents=...)` and `Bot(intents=...)` will error without it.
2. **`message_content` intent is privileged** — Without it, `message.content` is empty for messages not mentioning the bot.
3. **Override `on_message` in Bot? Call `process_commands`** — `await bot.process_commands(message)` at the end, or prefix commands break.
4. **Sync slash commands** — Call `await tree.sync()` (or `bot.tree.sync()`) in `setup_hook`. Don't sync on every `on_ready` (rate limits).
5. **Interaction timeout** — You have 3 seconds to respond. Use `defer()` for long operations.
6. **`defer()` then `followup.send()`** — After deferring, use `interaction.followup.send()` for the actual response.
7. **Ephemeral messages cannot be edited by others** — Only the bot that sent them can edit/delete.
8. **Views timeout after 180s by default** — Set `timeout=None` for persistent views and register them in `setup_hook`.
9. **`FFmpegPCMAudio` requires FFmpeg** — Install it system-wide; it's not a Python package.
10. **Avoid storing tokens in code** — Use environment variables or `.env` files.
