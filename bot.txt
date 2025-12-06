# -*- coding: utf-8 -*-
# 2025 终极完美版：转发永不丢按钮 + 自定义转发宣传文案 + 完整按钮增删改查全功能 + 排序功能 + 一行最多4个按钮
import json, logging, os
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InlineQueryResultCachedPhoto, InlineQueryResultCachedVideo, InlineQueryResultArticle, InputTextMessageContent
from telegram.ext import (
    ApplicationBuilder, CommandHandler, CallbackQueryHandler,
    MessageHandler, InlineQueryHandler, ContextTypes, ConversationHandler, filters
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
TOKEN = "8344376097:AAEx87w0zaXH8kMt8r7nKRgG7TtG49A-AXI" # ← 改成你的真实 Token
# ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←

DATA_FILE = "menus.json"
# 默认转发文案（如果菜单没单独设置，就用这个）
DEFAULT_FORWARD_CAPTION = "最新六合彩开奖直播\n实时特码｜精准杀庄｜永久域名\n点击下方按钮立即进入"
MAX_BUTTONS_PER_ROW = 4 # 一行最多4个按钮

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

# 数据迁移（兼容旧版）
migrated = False
for mid in list(menus.keys()):
    menu = menus[mid]
    if "rows" not in menu:
        buttons = menu.pop("buttons", [])
        menu["rows"] = []
        row = []
        for b in buttons:
            if "uid" not in b:
                b["uid"] = b.get("id", 0)
            row.append(b)
            if len(row) == MAX_BUTTONS_PER_ROW:
                menu["rows"].append(row)
                row = []
        if row:
            menu["rows"].append(row)
        max_uid = max([b["uid"] for row in menu["rows"] for b in row] + [-1])
        menu["next_uid"] = max_uid + 1
        # 重新分配每行的id
        for row in menu["rows"]:
            for i, b in enumerate(row):
                b["id"] = i
        migrated = True
    if "next_uid" not in menu:
        max_uid = max([b["uid"] for row in menu["rows"] for b in row] + [-1])
        menu["next_uid"] = max_uid + 1
        migrated = True
if migrated:
    save_menus()

# ==================== 键盘构建 ====================
def build_main_kb(mid):
    kb = []
    for row_buttons in menus[mid]["rows"]:
        if row_buttons:  # 跳过空行
            row = []
            for b in row_buttons:
                if b.get("subbuttons") and len(b["subbuttons"]) > 0:
                    row.append(InlineKeyboardButton(f"{b['name']} ▼", callback_data=f"open_{mid}*{b['uid']}"))
                else:
                    row.append(InlineKeyboardButton(b["name"], url=b.get("url", "https://t.me/")))
            kb.append(row)
    # 一键转发按钮（单独一行）- 改为空查询
    kb.append([InlineKeyboardButton("一键转发给好友", switch_inline_query="")])
    return InlineKeyboardMarkup(kb)

def build_sub_kb(subs, mid):
    kb = []
    row = []
    for s in subs:
        row.append(InlineKeyboardButton(s["name"], url=s.get("url", "https://t.me/")))
        if len(row) == MAX_BUTTONS_PER_ROW:
            kb.append(row)
            row = []
    if row:
        kb.append(row)
    # 返回 + 一键转发（单独一行）
    kb.append([InlineKeyboardButton("返回主菜单", callback_data=f"back*{mid}"), InlineKeyboardButton("一键转发给好友", switch_inline_query="")])
    return InlineKeyboardMarkup(kb)

# ==================== 所有命令 ====================
CREATING, EDITING = range(2)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("机器人已就绪！输入 /help 查看所有指令")

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = """*2025 终极版全功能指令（含完整增删改查 + 排序）*

/setmenu <菜单ID> → 创建一个新菜单（例如: /setmenu main），创建后会提示发送标题（文字、图片或视频）。
/edittitle <菜单ID> → 修改菜单标题（例如: /edittitle main），然后发送新标题（文字、图片或视频）。
/addbutton <菜单ID> <行号> <按钮名字> [链接] → 添加一级按钮到指定行（例如: /addbutton main 0 六合彩 https://example.com），链接可选，如果不填则默认为子菜单按钮。行号从0开始，如果等于当前行数则创建新行。
/addsubbutton <菜单ID> <行号> <父按钮ID> <名字> [链接] → 添加二级按钮（例如: /addsubbutton main 0 0 香港当前 https://xxx.com），链接可选。
/delbutton <菜单ID> <行号> <一级按钮ID> → 删除一级按钮及其所有二级按钮（例如: /delbutton main 0 1）。
/delsubbutton <菜单ID> <行号> <一级ID> <二级ID> → 删除指定二级按钮（例如: /delsubbutton main 0 0 2）。
/editbutton <菜单ID> <行号> <一级ID> <新名字> [新链接] → 修改一级按钮（例如: /editbutton main 0 0 新名字 https://new.com），新链接可选。
/editsubbutton <菜单ID> <行号> <一级ID> <二级ID> <新名字> [新链接] → 修改二级按钮（例如: /editsubbutton main 0 0 1 新名字 https://new.com），新链接可选。
/movebutton <菜单ID> <原行号> <一级ID> <新行号> <新位置> → 移动一级按钮到新行的指定位置（0-based，从0开始；可移到末尾，例如新位置等于当前该行按钮数；如果新行号等于当前行数则创建新行；每行最多4个，超出会报错）（例如: /movebutton main 0 3 1 0）。
/movesubbutton <菜单ID> <行号> <一级ID> <二级ID> <新位置> → 移动二级按钮到指定位置（0-based，从0开始；可移到末尾，例如新位置等于当前子按钮数；支持移到第二行或更后，如果子按钮多于4个）（例如: /movesubbutton main 0 0 1 3）。
/listbuttons <菜单ID> → 查看菜单的按钮列表，包括ID、名字、链接和顺序（例如: /listbuttons main）。
/sendmenu <菜单ID> → 发送可转发的菜单（例如: /sendmenu main）。
/setforward <菜单ID> <文案> → 设置转发宣传文案（换行用 \\n；例如: /setforward main 最新六合彩开奖直播\\n实时特码｜永久域名）。
/getforward <菜单ID> → 查看当前转发文案（例如: /getforward main）。
/listmenus → 查看所有菜单ID。
/delmenu <菜单ID> → 删除整个菜单（例如: /delmenu main）。

特点：转发永不丢按钮 + 文案完全自定义 + 按钮可随意增删改 + 排序（支持多行，灵活控制每行按钮数） + 一行最多4个按钮。"""
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
        "rows": [],
        "forward_caption": DEFAULT_FORWARD_CAPTION,
        "next_uid": 0
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
    if len(args) < 3:
        await update.message.reply_text("用法: /addbutton <ID> <行号> <按钮名字> [链接]")
        return
    mid = args[0]
    try:
        row_idx = int(args[1])
    except:
        await update.message.reply_text("行号必须是数字")
        return
    name = args[2]
    url = args[3] if len(args) > 3 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx < 0 or row_idx > len(rows):
        await update.message.reply_text("行号超出范围，请使用连续行号（新行用当前行数）")
        return
    if row_idx == len(rows):
        rows.append([])
    row = rows[row_idx]
    if len(row) >= MAX_BUTTONS_PER_ROW:
        await update.message.reply_text(f"该行已满，最多 {MAX_BUTTONS_PER_ROW} 个按钮")
        return
    uid = menus[mid]["next_uid"]
    menus[mid]["next_uid"] += 1
    btn = {"id": len(row), "uid": uid, "name": name, "subbuttons": []}
    if url: btn["url"] = url
    row.append(btn)
    save_menus()
    await update.message.reply_text(f"已添加一级按钮到行 {row_idx}：{name}（ID: {btn['id']}）")

# 添加二级按钮
async def addsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args
    if len(args) < 4:
        await update.message.reply_text("用法: /addsubbutton <菜单ID> <行号> <父按钮ID> <名字> [链接]")
        return
    mid = args[0]
    try:
        row_idx = int(args[1])
        pid = int(args[2])
    except:
        await update.message.reply_text("行号和ID必须是数字")
        return
    name = args[3]
    url = args[4] if len(args) > 4 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    parent = next((b for b in row if b["id"] == pid), None)
    if not parent:
        await update.message.reply_text("父按钮ID不存在")
        return
    sub = {"id": len(parent["subbuttons"]), "name": name}
    if url: sub["url"] = url
    parent["subbuttons"].append(sub)
    save_menus()
    await update.message.reply_text(f"已添加二级按钮：{name}")

# ==================== 删除一级按钮 ====================
async def delbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 3:
        await update.message.reply_text("用法: /delbutton <菜单ID> <行号> <一级按钮ID>")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        bid = int(context.args[2])
    except:
        await update.message.reply_text("行号和按钮ID必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    btn = next((b for b in row if b["id"] == bid), None)
    if not btn:
        await update.message.reply_text("未找到该一级按钮ID")
        return
    row.remove(btn)
    # 重新分配ID
    for i, b in enumerate(row):
        b["id"] = i
    if not row:
        rows.pop(row_idx)
    save_menus()
    await update.message.reply_text(f"已删除一级按钮（行 {row_idx} ID {bid}）及其所有二级按钮")

# ==================== 删除二级按钮 ====================
async def delsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 4:
        await update.message.reply_text("用法: /delsubbutton <菜单ID> <行号> <一级ID> <二级ID>")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        pid = int(context.args[2])
        sid = int(context.args[3])
    except:
        await update.message.reply_text("ID必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    parent = next((b for b in row if b["id"] == pid), None)
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

# ==================== 编辑一级按钮 ====================
async def editbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 4:
        await update.message.reply_text("用法: /editbutton <菜单ID> <行号> <一级ID> <新名字> [新链接]")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        bid = int(context.args[2])
    except:
        await update.message.reply_text("行号和按钮ID必须是数字")
        return
    new_name = context.args[3]
    new_url = context.args[4] if len(context.args) >= 5 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    btn = next((b for b in row if b["id"] == bid), None)
    if not btn:
        await update.message.reply_text("一级按钮不存在")
        return
    btn["name"] = new_name
    if new_url is not None:
        btn["url"] = new_url
    save_menus()
    await update.message.reply_text(f"一级按钮已更新为：{new_name}")

# ==================== 编辑二级按钮 ====================
async def editsubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 5:
        await update.message.reply_text("用法: /editsubbutton <菜单ID> <行号> <一级ID> <二级ID> <新名字> [新链接]")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        pid = int(context.args[2])
        sid = int(context.args[3])
    except:
        await update.message.reply_text("ID必须是数字")
        return
    new_name = context.args[4]
    new_url = context.args[5] if len(context.args) >= 6 else None
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    parent = next((b for b in row if b["id"] == pid), None)
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

# ==================== 新增：移动（排序）一级按钮 ====================
async def movebutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 5:
        await update.message.reply_text("用法: /movebutton <菜单ID> <原行号> <一级ID> <新行号> <新位置> (新位置从0开始，可等于当前该行按钮数以移到末尾)")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        bid = int(context.args[2])
        new_row_idx = int(context.args[3])
        new_pos = int(context.args[4])
    except:
        await update.message.reply_text("行号、ID和位置必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("原行号不存在")
        return
    row = rows[row_idx]
    btn = next((b for b in row if b["id"] == bid), None)
    if not btn:
        await update.message.reply_text("一级按钮不存在")
        return
    row.remove(btn)
    # 重新分配原行ID
    for i, b in enumerate(row):
        b["id"] = i
    if new_row_idx < 0 or new_row_idx > len(rows):
        await update.message.reply_text("新行号超出范围，请使用连续行号（新行用当前行数）")
        return
    if new_row_idx == len(rows):
        rows.append([])
    new_row = rows[new_row_idx]
    if len(new_row) + 1 > MAX_BUTTONS_PER_ROW:
        await update.message.reply_text("移动后新行会超出最多4个按钮")
        return
    if new_pos < 0 or new_pos > len(new_row):
        await update.message.reply_text("新位置超出范围")
        return
    # 插入到新位置（允许new_pos == len(new_row) 以移到末尾）
    new_row.insert(new_pos, btn)
    # 重新分配新行ID
    for i, b in enumerate(new_row):
        b["id"] = i
    if not row:
        rows.pop(row_idx)
    save_menus()
    await update.message.reply_text(f"已将一级按钮（原行 {row_idx} ID {bid}）移动到行 {new_row_idx} 位置 {new_pos}")

# ==================== 新增：移动（排序）二级按钮 ====================
async def movesubbutton(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 5:
        await update.message.reply_text("用法: /movesubbutton <菜单ID> <行号> <一级ID> <二级ID> <新位置> (新位置从0开始，可等于当前子按钮数以移到末尾)")
        return
    mid = context.args[0]
    try:
        row_idx = int(context.args[1])
        pid = int(context.args[2])
        sid = int(context.args[3])
        new_pos = int(context.args[4])
    except:
        await update.message.reply_text("ID和位置必须是数字")
        return
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    rows = menus[mid]["rows"]
    if row_idx >= len(rows) or row_idx < 0:
        await update.message.reply_text("行号不存在")
        return
    row = rows[row_idx]
    parent = next((b for b in row if b["id"] == pid), None)
    if not parent:
        await update.message.reply_text("一级按钮不存在")
        return
    subbuttons = parent["subbuttons"]
    sub = next((s for s in subbuttons if s["id"] == sid), None)
    if not sub:
        await update.message.reply_text("二级按钮不存在")
        return
    subbuttons.remove(sub)
    if new_pos < 0 or new_pos > len(subbuttons):
        await update.message.reply_text("新位置超出范围")
        return
    # 插入到新位置（允许new_pos == len(subbuttons) 以移到末尾）
    subbuttons.insert(new_pos, sub)
    # 重新分配ID
    for i, s in enumerate(subbuttons):
        s["id"] = i
    save_menus()
    await update.message.reply_text(f"已将二级按钮（原ID {sid}）移动到位置 {new_pos}")

# ==================== 新增：查看按钮列表 ====================
async def listbuttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("用法: /listbuttons <菜单ID>")
        return
    mid = context.args[0]
    if mid not in menus:
        await update.message.reply_text("菜单不存在")
        return
    text = f"*菜单 {mid} 按钮列表：*\n\n"
    for r_idx, row in enumerate(menus[mid]["rows"]):
        text += f"行 {r_idx}:\n"
        for b in row:
            text += f"ID {b['id']}: {b['name']}"
            if "url" in b:
                text += f" ({b['url']})"
            text += "\n"
            if b["subbuttons"]:
                text += " 子按钮：\n"
                for s in b["subbuttons"]:
                    text += f" - ID {s['id']}: {s['name']}"
                    if "url" in s:
                        text += f" ({s['url']})"
                    text += "\n"
            text += "\n"
        text += "\n"
    await update.message.reply_text(text, parse_mode="Markdown")

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
def find_btn(mid, uid):
    for row in menus[mid]["rows"]:
        for b in row:
            if b["uid"] == uid:
                return b
    return None

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    data = q.data
    if data.startswith("open_"):
        temp = data[5:]
        mid, uid_str = temp.split("*")
        uid = int(uid_str)
        btn = find_btn(mid, uid)
        if btn:
            await q.edit_message_reply_markup(reply_markup=build_sub_kb(btn["subbuttons"], mid))
    elif data.startswith("back*"):
        mid = data[5:]
        await q.edit_message_reply_markup(reply_markup=build_main_kb(mid))

# 一键转发（inline模式）- 关键修复
async def inlinequery(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.inline_query.query.strip()
    results = []
    
    logger.info(f"收到内联查询: '{query}', 可用菜单: {list(menus.keys())}")
    
    # 如果查询为空，显示所有可用菜单
    if not query:
        if menus:
            for mid in menus.keys():
                # 显示所有菜单的快捷入口
                results.append(
                    InlineQueryResultArticle(
                        id=f"menu_{mid}",
                        title=f"📋 菜单 {mid}",
                        description=f"点击发送菜单 {mid}",
                        input_message_content=InputTextMessageContent(
                            f"菜单 {mid} 已准备好发送！"
                        )
                    )
                )
        else:
            results.append(
                InlineQueryResultArticle(
                    id="no_menus",
                    title="❌ 暂无可用菜单",
                    description="请先创建菜单",
                    input_message_content=InputTextMessageContent("暂无可用菜单，请使用 /setmenu 创建")
                )
            )
    else:
        # 尝试查找菜单
        mid = query
        if mid not in menus:
            # 也尝试将查询作为字符串处理，因为JSON键是字符串
            mid_str = str(query)
            if mid_str in menus:
                mid = mid_str
            else:
                available = list(menus.keys())
                results.append(
                    InlineQueryResultArticle(
                        id="not_found",
                        title=f"❌ 菜单 '{query}' 不存在",
                        description=f"可用菜单: {', '.join(available) if available else '无'}",
                        input_message_content=InputTextMessageContent(
                            f"菜单 '{query}' 不存在\n可用菜单: {', '.join(available) if available else '无'}"
                        )
                    )
                )
        
        # 如果找到了菜单
        if mid in menus:
            m = menus[mid]
            kb = build_main_kb(mid)
            caption = m.get("forward_caption", DEFAULT_FORWARD_CAPTION)
            
            try:
                if m["title_type"] == "photo":
                    results.append(
                        InlineQueryResultCachedPhoto(
                            id=f"photo_{mid}",
                            photo_file_id=m["title_content"],
                            title=f"📸 菜单 {mid}",
                            caption=caption,
                            reply_markup=kb
                        )
                    )
                elif m["title_type"] == "video":
                    results.append(
                        InlineQueryResultCachedVideo(
                            id=f"video_{mid}",
                            video_file_id=m["title_content"],
                            title=f"🎥 菜单 {mid}",
                            caption=caption,
                            reply_markup=kb
                        )
                    )
                else:
                    # 文本菜单
                    title_content = m.get("title_content", f"菜单 {mid}")
                    results.append(
                        InlineQueryResultArticle(
                            id=f"text_{mid}",
                            title=f"📝 菜单 {mid}",
                            description="点击发送此菜单",
                            input_message_content=InputTextMessageContent(
                                f"{title_content}\n\n{caption}"
                            ),
                            reply_markup=kb
                        )
                    )
                logger.info(f"成功创建菜单 {mid} 的内联结果")
            except Exception as e:
                logger.error(f"内联查询处理失败: {e}")
                # 降级为文本模式
                results.append(
                    InlineQueryResultArticle(
                        id=f"fallback_{mid}",
                        title=f"📝 菜单 {mid} (文本模式)",
                        description="点击发送此菜单",
                        input_message_content=InputTextMessageContent(
                            f"菜单 {mid}\n\n{caption}"
                        ),
                        reply_markup=kb
                    )
                )
    
    # 关键：必须总是返回响应
    try:
        await update.inline_query.answer(results, cache_time=1, is_personal=True)
        logger.info(f"已返回 {len(results)} 个内联结果")
    except Exception as e:
        logger.error(f"内联查询响应失败: {e}")
        # 即使出错也要尝试返回空结果
        await update.inline_query.answer([], cache_time=1)

# ==================== 启动 ====================
def main():
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
    # 增删改命令
    app.add_handler(CommandHandler("delbutton", delbutton))
    app.add_handler(CommandHandler("delsubbutton", delsubbutton))
    app.add_handler(CommandHandler("editbutton", editbutton))
    app.add_handler(CommandHandler("editsubbutton", editsubbutton))
    # 新增排序命令
    app.add_handler(CommandHandler("movebutton", movebutton))
    app.add_handler(CommandHandler("movesubbutton", movesubbutton))
    # 新增查看按钮列表命令
    app.add_handler(CommandHandler("listbuttons", listbuttons))
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
    
    logger.info("【2025终极完美版 + 完整按钮增删改查 + 排序 + 一行最多4个按钮】已启动！")
    logger.info(f"当前菜单数量: {len(menus)}")
    logger.info(f"菜单列表: {list(menus.keys())}")
    
    app.run_polling(drop_pending_updates=True)

if __name__ == "__main__":
    main()
