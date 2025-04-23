
import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime, timedelta
import threading
from plyer import notification
import time

tasks = []

# Function to get the time formatted
def get_time(hour_var, min_var, ampm_var):
    hour = hour_var.get()
    minute = min_var.get()
    ampm = ampm_var.get()
    if hour and minute and ampm:
        return f"{hour}:{minute} {ampm}"
    return None

# Add Task Function
def add_task():
    task_name = entry_task.get()
    priority = priority_var.get()
    start_time = get_time(start_hour, start_min, start_ampm)
    end_time = get_time(end_hour, end_min, end_ampm)
    
    if not task_name:
        messagebox.showwarning("Missing Info", "Please enter task name.")
        return
    
    try:
        start_obj = datetime.strptime(start_time, "%I:%M %p")
        end_obj = datetime.strptime(end_time, "%I:%M %p")
        
        if start_obj >= end_obj:
            messagebox.showwarning("Time Error", "End time must be after start time.")
            return
        
        task = {
            "task": task_name,
            "priority": priority,
            "start": start_obj.strftime("%I:%M %p"),
            "end": end_obj.strftime("%I:%M %p"),
            "done": False,
            "notified_start": False,
            "notified_end": False
        }

        tasks.append(task)
        
        # Color code based on priority
        if priority == "High":
            display = f"[🔴 {priority.upper()}] {task_name} | {task['start']} ➡ {task['end']}"
            listbox.insert(tk.END, display)
            listbox.itemconfig(tk.END, {'fg': 'red'})
        elif priority == "Medium":
            display = f"[🟠 {priority.upper()}] {task_name} | {task['start']} ➡ {task['end']}"
            listbox.insert(tk.END, display)
            listbox.itemconfig(tk.END, {'fg': 'orange'})
        else:
            display = f"[🟢 {priority.upper()}] {task_name} | {task['start']} ➡ {task['end']}"
            listbox.insert(tk.END, display)
            listbox.itemconfig(tk.END, {'fg': 'green'})

        entry_task.delete(0, tk.END)

    except ValueError:
        messagebox.showwarning("Invalid Time", "Please enter valid start and end times.")

# Mark Task as Done
def mark_done():
    try:
        index = listbox.curselection()[0]
        tasks[index]["done"] = True
        listbox.itemconfig(index, {'fg': 'green'})
        show_motivation(tasks[index]["task"])
    except:
        pass

# Delete Task Function
def delete_task():
    try:
        index = listbox.curselection()[0]
        listbox.delete(index)
        del tasks[index]
    except:
        pass

# Motivation Notifications after Task is Done
def show_motivation(task):
    now = datetime.now()
    hour = now.hour
    
    if 5 <= hour < 12:
        quote = "🌞 Rise and shine! The world is yours."
    elif 12 <= hour < 17:
        quote = "☀️ Keep pushing, you're doing amazing!"
    elif 17 <= hour < 21:
        quote = "🌇 You’ve made progress today. Be proud."
    else:
        quote = "🌙 Rest, recharge, and rise stronger tomorrow."

    notification.notify(
        title="✅ Task Completed!",
        message=f"You completed: {task}\n{quote}",
        timeout=5
    )

# Task Start and End Notifications
def notify_start(task):
    notification.notify(
        title="⏰ Time to Start!",
        message=f"Start: {task}\n“You got this!” 🚀",
        timeout=5
    )

def notify_end(task, done):
    if done:
        notification.notify(
            title="✅ Finished On Time!",
            message=f"Great job finishing: {task}! 🎉",
            timeout=5
        )
    else:
        notification.notify(
            title="❌ Time's Up!",
            message=f"Time's up for: {task}\n“Try to catch up or reschedule.” 🌧️",
            timeout=5
        )

# Background thread to check tasks
def check_tasks():
    while True:
        now = datetime.now().strftime("%I:%M %p")
        for task in tasks:
            if task["start"] == now and not task["notified_start"]:
                notify_start(task["task"])
                task["notified_start"] = True
            if task["end"] == now and not task["notified_end"]:
                notify_end(task["task"], task["done"])
                task["notified_end"] = True
        time.sleep(60)

# Pomodoro Timer
pomodoro_running = False
pomodoro_timer = None

def start_pomodoro():
    global pomodoro_running, pomodoro_timer
    
    if pomodoro_running:
        return
    
    pomodoro_running = True
    work_duration = timedelta(minutes=25)
    break_duration = timedelta(minutes=5)

    def run_pomodoro():
        work_end_time = datetime.now() + work_duration
        notification.notify(
            title="⏰ Start Work!",
            message="Work session started! 📝",
            timeout=5
        )
        time.sleep(work_duration.total_seconds())
        
        notification.notify(
            title="⏸ Break Time!",
            message="Take a 5-minute break! 🍵",
            timeout=5
        )
        time.sleep(break_duration.total_seconds())

        notification.notify(
            title="⏰ Back to Work!",
            message="Work session resumed! 🚀",
            timeout=5
        )
        pomodoro_running = False
        
    threading.Thread(target=run_pomodoro, daemon=True).start()

# Review Tasks Function
def review_tasks():
    completed = [t for t in tasks if t["done"]]
    missed = [
        t for t in tasks
        if not t["done"] and datetime.strptime(t["end"], "%I:%M %p") < datetime.now()
    ]
    ongoing = [
        t for t in tasks
        if not t["done"] and datetime.strptime(t["end"], "%I:%M %p") >= datetime.now()
    ]
    
    summary = (
        f"✅ Completed: {len(completed)}\n"
        f"🕒 Ongoing: {len(ongoing)}\n"
        f"❌ Missed: {len(missed)}\n\n"
        "Take a moment to review and adjust priorities."
    )
    messagebox.showinfo("🔄 Daily Review", summary)

# GUI Setup
app = tk.Tk()
app.title("📝 Pro To-Do with Priorities & Emo Power")

frame = tk.Frame(app)
frame.pack(pady=10)

entry_task = tk.Entry(frame, width=30)
entry_task.grid(row=0, column=0, columnspan=5, padx=5)

# Priority Dropdown
priority_var = tk.StringVar()
priority_options = ["High", "Medium", "Low"]
priority_menu = ttk.Combobox(frame, textvariable=priority_var, values=priority_options, width=7)
priority_menu.set("Medium")
priority_menu.grid(row=0, column=5)

# Time Pickers
hours = [f"{i:02}" for i in range(1, 13)]
minutes = [f"{i:02}" for i in range(0, 60)]
ampm = ["AM", "PM"]

# Start Time Selectors
start_hour = ttk.Combobox(frame, values=hours, width=3)
start_min = ttk.Combobox(frame, values=minutes, width=3)
start_ampm = ttk.Combobox(frame, values=ampm, width=3)
start_hour.set("HH")
start_min.set("MM")
start_ampm.set("AM/PM")

start_hour.grid(row=1, column=0)
start_min.grid(row=1, column=1)
start_ampm.grid(row=1, column=2)

# End Time Selectors
end_hour = ttk.Combobox(frame, values=hours, width=3)
end_min = ttk.Combobox(frame, values=minutes, width=3)
end_ampm = ttk.Combobox(frame, values=ampm, width=3)
end_hour.set("HH")
end_min.set("MM")
end_ampm.set("AM/PM")

end_hour.grid(row=1, column=3)
end_min.grid(row=1, column=4)
end_ampm.grid(row=1, column=5)

# Add Task Button
tk.Button(frame, text="➕ Add Task", command=add_task).grid(row=1, column=6, padx=5)

listbox = tk.Listbox(app, width=70, height=10)
listbox.pack(pady=10)

tk.Button(app, text="✅ Mark Done", command=mark_done).pack(pady=2)
tk.Button(app, text="🗑️ Delete Task", command=delete_task).pack(pady=2)
tk.Button(app, text="🔄 Review Tasks", command=review_tasks).pack(pady=5)

# Pomodoro Button
tk.Button(app, text="🍅 Start Focus Session", command=start_pomodoro).pack(pady=5)

# Background Thread for Task Check
threading.Thread(target=check_tasks, daemon=True).start()

app.mainloop()
