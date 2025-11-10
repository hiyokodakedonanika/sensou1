import discord
from discord.ext import commands, tasks
from discord import app_commands
import asyncio
import datetime

intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)

# 待機中のユーザー
waiting_users = []

# クールタイム管理
cooldowns = {}  # user_id: datetime

# 論争スレッドの最終発言記録
thread_activity = {}  # thread_id: datetime


class MatchButton(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="⚔️ 論争マッチングに参加！", style=discord.ButtonStyle.blurple)
    async def match_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        user = interaction.user
        now = datetime.datetime.utcnow()

        # クールタイム中かチェック
        if user.id in cooldowns:
            diff = (now - cooldowns[user.id]).total_seconds()
            if diff < 3600:  # 1時間 = 3600秒
                remaining = int((3600 - diff) / 60)
                await interaction.response.send_message(
                    f"⏳ クールタイム中です！あと {remaining} 分待ってね。", ephemeral=True
                )
                return

        # すでに待機中ならスキップ
        if user in waiting_users:
            await interaction.response.send_message("🕒 すでに待機中です！", ephemeral=True)
            return

        waiting_users.append(user)
        await interaction.response.send_message("💭 待機リストに入りました！マッチングを待っています...", ephemeral=True)

        # 2人揃ったらマッチング
        if len(waiting_users) >= 2:
            user1 = waiting_users.pop(0)
            user2 = waiting_users.pop(0)

            # クールタイム開始
            cooldowns[user1.id] = now
            cooldowns[user2.id] = now

            # 論争スレッド作成
            channel = interaction.channel
            thread_name = f"⚔️ 論争 - {user1.name} vs {user2.name}"
            thread = await channel.create_thread(name=thread_name, type=discord.ChannelType.public_thread)

            await thread.send(
                f"🔥 **論争開始！**\n{user1.mention} vs {user2.mention}\n\n🕒 3時間発言がなければ自動削除されます。"
            )

            # 最終活動時間を記録
            thread_activity[thread.id] = datetime.datetime.utcnow()


# 発言を監視して最終アクティビティを更新
@bot.event
async def on_message(message: discord.Message):
    if message.author.bot:
        return

    if message.channel.type == discord.ChannelType.public_thread:
        if message.channel.id in thread_activity:
            thread_activity[message.channel.id] = datetime.datetime.utcnow()

    await bot.process_commands(message)


# 定期的にスレッドをチェックして削除
@tasks.loop(minutes=5)
async def check_inactive_threads():
    now = datetime.datetime.utcnow()
    to_delete = []

    for thread_id, last_active in list(thread_activity.items()):
        if (now - last_active).total_seconds() > 10800:  # 3時間 = 10800秒
            to_delete.append(thread_id)

    for thread_id in to_delete:
        thread = bot.get_channel(thread_id)
        if thread:
            await thread.send("💤 3時間発言がなかったため、この論争スレッドは自動で削除されます。")
            await asyncio.sleep(5)
            await thread.delete()
        del thread_activity[thread_id]


@bot.tree.command(name="論争マッチング", description="論争マッチング用ボタンを表示します")
async def match_command(interaction: discord.Interaction):
    view = MatchButton()
    await interaction.response.send_message("🎯 ボタンを押して論争マッチングに参加しよう！", view=view)


@bot.event
async def on_ready():
    await bot.tree.sync()
    check_inactive_threads.start()
    print(f"✅ ログイン完了: {bot.user}")


bot.run("YOUR_BOT_TOKEN_HERE")
