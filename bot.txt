# -*- coding: utf-8 -*-
# 2025 终极完美版：转发永不丢按钮 + 自定义转发宣传文案 + 完整按钮增删改查全功能
import json, logging, os
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InlineQueryResultCachedPhoto, InlineQueryResultCachedVideo, InlineQueryResultArticle, InputTextMessageContent
from telegram.ext import (
    ApplicationBuilder, CommandHandler, CallbackQueryHandler,
    MessageHandler, InlineQueryHandler, ContextTypes, ConversationHandler, filters
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
TOKEN = "8344376097:AAEx87w0zaXH8kMt8r7nKRgG7TtG49A-AXI"  # ← 改成你的真实 Token
# ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←

DATA_FILE = "menus.json"
# 默认转发文案（如果菜单没单独设置，就用这个）
DEFAULT_FORWARD_CAPTION = "最新六合彩开奖直播\n实时特码｜精准杀庄｜永久域名\n点击下方按钮立即进入"

if os.path.exists(DATA_FILE):
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            menus = json.load(f)
    except:
        menus = {}
else:
    menus = {}

def save_menus():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(menus, f, ensure_ascii=False, indent=2)

# ==================== 键盘构建 ====================
def build_main_kb(mid):
    kb = []
    for b in menus[mid]["buttons"]:
        if b.get("subbuttons") and len(b["subbuttons"]) > 0:
            kb.append([InlineKeyboardButton(f"{b['name']} ▼", callback_data=f"open_{mid}*{b['id']}")])
        else:
            kb.append([InlineKeyboardButton(b["name"], url=b.get("url", "https://t.me/"))])
   
    # 一键转发按钮
    kb.append([InlineKeyboardButton("一键转发给好友", switch_inline_query_current_chat=mid)])
    return InlineKeyboardMarkup(kb)

def build_sub_kb(subs, mid):
    kb = []
    row = []
    for s in subs:
        row.append(InlineKeyboardButton(s["name"], url=s.get("url", "https://t.me/")))
        if len(row) == 2:
            kb.append(row)
            row = []
    if row:
        kb.append(row)
    # 返回 + 一键转发
    kb.append([InlineKeyboardButton("返回主菜单", callback_data=f"back*{mid}")])
    kb.append([InlineKeyboardButton("一键转发给好友", switch_inline_query_current_chat=mid)])
    return InlineKeyboardMarkup(kb)

# ==================== 所有命令 ====================
CREATING, EDITING = range(2)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("机器人已就绪！输入 /help 查看所有指令")

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = """*2025 终极版全功能指令（含完整增删改查）*

/setmenu main → 创建新菜单
/edittitle main → 修改标题（文字/图/视频）
/addbutton main 六合彩 → 添加一级按钮
/addsubbutton main 0 香港当前 https://xxx.com → 添加二级按钮

/delbutton main 1 → 删除一级按钮（连二级一起删）
/delsubbutton main 0 2 → 删除指定二级按钮
/editbutton main 0 新名字 https://new.com → 修改一级按钮名字和链接
/editsubbutton main 0 1 新名字 https://new.com → 修改二级按钮

/sendmenu main → 发送可转发菜单
/setforward main 最新六合彩开奖直播\\n实时特码｜永久域名 → 设置转发宣传文案
/getforward main → 查看当前转发文案
/listmenus → 查看所有菜单ID
/delmenu main → 删除整个菜单

转发永不丢按钮 + 文案完全自定义 + 按钮可随意增删改"""
    await update.message.reply_text(help_text, parse_mode="Markdown")

# 创建菜单
async def setmenu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /setmenu <ID>")
        return ConversationHandler.END
    mid = context.args[0]
    if mid in menus:
        await update.message.reply_text("ID已存在，请换一个")
        return ConversationHandler.END
    menus[mid] = {
        "title_type": None,
        "title_content": None,
        "buttons": [],
        "forward_caption": DEFAULT_FORWARD_CAPTION
    }
    save_menus()
    context.user_data["new"] = mid
    await update.message.reply_text(f"「{mid}」创建成功！\n现在请发送标题内容（文字或图片或视频）")
    return CREATING

async def receive_title(update: Update, context: ContextTypes.DEFAULT_TYPE):
    mid = context.user_data.get("new")
    if not mid: return ConversationHandler.END
    if update.message.photo:
        menus[mid]["title_type"] = "photo"
        menus[mid]["title_content"] = update.message.photo[-1].file_id
    elif update.message.video:
        menus[mid]["title_type"] = "video"
        menus[mid]["title_content"] = update.message.video.file_id
    elif update.message.text:
        menus[mid]["title_type"] = "text"
        menus[mid]["title_content"] = update.message.text
    else:
        await update.message.reply_text("不支持的格式，请发文字/图片/视频")
        return CREATING
    save_menus()
    await update.message.reply_text("标题设置完成！现在可以用 /addbutton 添加按钮")
    return ConversationHandler.END

# 修改标题
async def edittitle(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /edittitle <ID>")
        return ConversationHandler.END
    mid = context.args[0]
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return ConversationHandler.END
    context.user_data["edit"] = mid
    await update.message.reply_text("请发送新的标题（文字/图片/视频）")
    return EDITING

async def receive_new_title(update: Update, context: ContextTypes.DEFAULT_TYPE):
    mid = context.user_data.get("edit")
    if not mid: return ConversationHandler.END
    if update.message.photo:
        menus[mid]["title_type"] = "photo"
        menus[mid]["title_content"] = update.message.photo[-1].file_id
    elif update.message.video:
        menus[mid]["title_type"] = "video"
        menus[mid]["title_content"] = update.message.video.file_id
    elif update.message.text:
        menus[mid]["title_type"] = "text"
        menus[mid]["title_content"] = update.message.text
    save_menus()
    await update.message.reply_text("标题已更新！")
    return ConversationHandler.END

# 添加一级按钮
async def addbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("用法: /addbutton <ID> <按钮名字> [链接]")
        return
    mid, name = args[0], args[1]
    url = args[2] if len(args) > 2 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    btn = {"id": len(menus[mid]["buttons"]), "name": name, "subbuttons": []}
    if url: btn["url"] = url
    menus[mid]["buttons"].append(btn)
    save_menus()
    await update.message.reply_text(f"已添加一级按钮：{name}（ID: {btn['id']}）")

# 添加二级按钮
async def addsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args
    if len(args) < 3:
        await update.message.reply_text("用法: /addsubbutton <菜单ID> <父按钮ID> <名字> [链接]")
        return
    mid, pid, name = args[0], int(args[1]), args[2]
    url = args[3] if len(args) > 3 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    parent = next((b for b in menus[mid]["buttons"] if b["id"] == pid), None)
    if not parent:
        await update.message.reply_text("父按钮ID不存在")
        return
    sub = {"id": len(parent["subbuttons"]), "name": name}
    if url: sub["url"] = url
    parent["subbuttons"].append(sub)
    save_menus()
    await update.message.reply_text(f"已添加二级按钮：{name}")

# ==================== 新增：删除一级按钮 ====================
async def delbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 2:
        await update.message.reply_text("用法: /delbutton <菜单ID> <一级按钮ID>")
        return
    mid = context.args[0]
    try:
        bid = int(context.args[1])
    except:
        await update.message.reply_text("按钮ID必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    original_len = len(menus[mid]["buttons"])
    menus[mid]["buttons"] = [b for b in menus[mid]["buttons"] if b["id"] != bid]
    if len(menus[mid]["buttons"]) == original_len:
        await update.message.reply_text("未找到该一级按钮ID")
        return
    # 重新分配ID（保持顺序美观）
    for i, b in enumerate(menus[mid]["buttons"]):
        b["id"] = i
    save_menus()
    await update.message.reply_text(f"已删除一级按钮（ID {bid}）及其所有二级按钮")

# ==================== 新增：删除二级按钮 ====================
async def delsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 3:
        await update.message.reply_text("用法: /delsubbutton <菜单ID> <一级ID> <二级ID>")
        return
    mid = context.args[0]
    try:
        pid = int(context.args[1])
        sid = int(context.args[2])
    except:
        await update.message.reply_text("ID必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    parent = next((b for b in menus[mid]["buttons"] if b["id"] == pid), None)
    if not parent:
        await update.message.reply_text("一级按钮不存在")
        return
    original_len = len(parent["subbuttons"])
    parent["subbuttons"] = [s for s in parent["subbuttons"] if s["id"] != sid]
    if len(parent["subbuttons"]) == original_len:
        await update.message.reply_text("未找到该二级按钮ID")
        return
    # 重新分配二级ID
    for i, s in enumerate(parent["subbuttons"]):
        s["id"] = i
    save_menus()
    await update.message.reply_text(f"已删除二级按钮（ID {sid}）")

# ==================== 新增：编辑一级按钮 ====================
async def editbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 3:
        await update.message.reply_text("用法: /editbutton <菜单ID> <一级ID> <新名字> [新链接]")
        return
    mid = context.args[0]
    try:
        bid = int(context.args[1])
    except:
        await update.message.reply_text("按钮ID必须是数字")
        return
    new_name = context.args[2]
    new_url = context.args[3] if len(context.args) >= 4 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    btn = next((b for b in menus[mid]["buttons"] if b["id"] == bid), None)
    if not btn:
        await update.message.reply_text("一级按钮不存在")
        return
    btn["name"] = new_name
    if new_url is not None:
        btn["url"] = new_url
    save_menus()
    await update.message.reply_text(f"一级按钮已更新为：{new_name}")

# ==================== 新增：编辑二级按钮 ====================
async def editsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 4:
        await update.message.reply_text("用法: /editsubbutton <菜单ID> <一级ID> <二级ID> <新名字> [新链接]")
        return
    mid = context.args[0]
    try:
        pid = int(context.args[1])
        sid = int(context.args[2])
    except:
        await update.message.reply_text("ID必须是数字")
        return
    new_name = context.args[3]
    new_url = context.args[4] if len(context.args) >= 5 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    parent = next((b for b in menus[mid]["buttons"] if b["id"] == pid), None)
    if not parent:
        await update.message.reply_text("一级按钮不存在")
        return
    sub = next((s for s in parent["subbuttons"] if s["id"] == sid), None)
    if not sub:
        await update.message.reply_text("二级按钮不存在")
        return
    sub["name"] = new_name
    if new_url is not None:
        sub["url"] = new_url
    save_menus()
    await update.message.reply_text(f"二级按钮已更新为：{new_name}")

# 自定义转发文案
async def setforward(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 2:
        await update.message.reply_text("用法: /setforward <菜单ID> <文案>\n换行用 \\n\n示例：\n/setforward main 最新六合彩开奖直播\\n实时特码｜永久域名")
        return
    mid = context.args[0]
    text = " ".join(context.args[1:])
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    menus[mid]["forward_caption"] = text.replace("\\n", "\n")
    save_menus()
    await update.message.reply_text(f"「{mid}」转发文案已更新：\n\n{menus[mid]['forward_caption']}")

async def getforward(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /getforward <菜单ID>")
        return
    mid = context.args[0]
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    caption = menus[mid].get("forward_caption", DEFAULT_FORWARD_CAPTION)
    await update.message.reply_text(f"「{mid}」当前转发文案：\n\n{caption}")

# 发送菜单
async def sendmenu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /sendmenu <ID>")
        return
    mid = context.args[0]
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    m = menus[mid]
    kb = build_main_kb(mid)
    caption = m.get("forward_caption", DEFAULT_FORWARD_CAPTION)
    try:
        if m["title_type"] == "photo":
            await update.message.reply_photo(
                photo=m["title_content"],
                caption=caption,
                reply_markup=kb
            )
        elif m["title_type"] == "video":
            await update.message.reply_video(
                video=m["title_content"],
                caption=caption,
                reply_markup=kb
            )
        else:
            await update.message.reply_text(
                m.get("title_content", "菜单") + "\n" + caption,
                reply_markup=kb
            )
    except Exception as e:
        logger.error(f"发送失败: {e}")
        await update.message.reply_text("发送失败，媒体可能已过期")

async def listmenus(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not menus:
        await update.message.reply_text("暂无菜单")
    else:
        text = "*现有菜单ID：*\n" + "\n".join(f"• {m}" for m in menus.keys())
        await update.message.reply_text(text, parse_mode="Markdown")

async def delmenu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /delmenu <ID>")
        return
    mid = context.args[0]
    if mid in menus:
        del menus[mid]
        save_menus()
        await update.message.reply_text(f"已删除菜单：{mid}")
    else:
        await update.message.reply_text("菜单不存在")

# 按钮点击处理
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    data = q.data
    if data.startswith("open_"):
        _, temp = data.split("_", 1)
        mid, bid = temp.split("*")
        bid = int(bid)
        subbuttons = next(b for b in menus[mid]["buttons"] if b["id"] == bid)["subbuttons"]
        await q.edit_message_reply_markup(reply_markup=build_sub_kb(subbuttons, mid))
    elif data.startswith("back*"):
        mid = data[5:]
        await q.edit_message_reply_markup(reply_markup=build_main_kb(mid))

# 一键转发（inline模式）
async def inlinequery(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.inline_query.query.strip()
    if not query or query not in menus:
        return
    mid = query
    m = menus[mid]
    kb = build_main_kb(mid)
    caption = m.get("forward_caption", DEFAULT_FORWARD_CAPTION)
    results = []
    if m["title_type"] == "photo":
        results.append(
            InlineQueryResultCachedPhoto(
                id=mid + "_photo",
                photo_file_id=m["title_content"],
                title="点击分享此菜单",
                caption=caption,
                reply_markup=kb
            )
        )
    elif m["title_type"] == "video":
        results.append(
            InlineQueryResultCachedVideo(
                id=mid + "_video",
                video_file_id=m["title_content"],
                title="点击分享此菜单",
                caption=caption,
                reply_markup=kb
            )
        )
    else:
        results.append(
            InlineQueryResultArticle(
                id=mid + "_text",
                title="点击分享此菜单",
                input_message_content=InputTextMessageContent(
                    m.get("title_content", "菜单") + "\n\n" + caption
                ),
                reply_markup=kb
            )
        )
    await update.inline_query.answer(results, cache_time=0)

# ==================== 启动 ====================
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("help", help_cmd))
app.add_handler(CommandHandler("sendmenu", sendmenu))
app.add_handler(CommandHandler("listmenus", listmenus))
app.add_handler(CommandHandler("delmenu", delmenu))
app.add_handler(CommandHandler("addbutton", addbutton))
app.add_handler(CommandHandler("addsubbutton", addsubbutton))
app.add_handler(CommandHandler("setforward", setforward))
app.add_handler(CommandHandler("getforward", getforward))

# 新增的四个命令注册
app.add_handler(CommandHandler("delbutton", delbutton))
app.add_handler(CommandHandler("delsubbutton", delsubbutton))
app.add_handler(CommandHandler("editbutton", editbutton))
app.add_handler(CommandHandler("editsubbutton", editsubbutton))

app.add_handler(ConversationHandler(
    entry_points=[CommandHandler("setmenu", setmenu)],
    states={CREATING: [MessageHandler(filters.PHOTO | filters.VIDEO | filters.TEXT & ~filters.COMMAND, receive_title)]},
    fallbacks=[]
))

app.add_handler(ConversationHandler(
    entry_points=[CommandHandler("edittitle", edittitle)],
    states={EDITING: [MessageHandler(filters.PHOTO | filters.VIDEO | filters.TEXT & ~filters.COMMAND, receive_new_title)]},
    fallbacks=[]
))

app.add_handler(CallbackQueryHandler(button))
app.add_handler(InlineQueryHandler(inlinequery))

logger.info("【2025终极完美版 + 完整按钮增删改查】已启动！完美运行！")
app.run_polling(drop_pending_updates=True)