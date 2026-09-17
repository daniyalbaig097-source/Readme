# Readme
Educational app
from kivy.app import App
from kivy.uix.screenmanager import ScreenManager, Screen
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.scrollview import ScrollView
from kivy.uix.progressbar import ProgressBar
from kivy.storage.jsonstore import JsonStore
from kivy.metrics import dp

store = JsonStore("al_hadi_data.json")

QUESTIONS = {
    "Computer Science": {
        "Programming": [
            ("What language is this prototype written in?", ["Python","Java","C++","PHP"], 0),
            ("Which symbol starts a Python comment?", ["//","#","<!--","/*"], 1),
            ("Which type stores True or False?", ["String","Boolean","Float","List"], 1),
        ],
        "Computer Basics": [
            ("What does CPU stand for?", ["Central Processing Unit","Computer Personal Unit","Central Program Utility","Control Processing User"], 0),
            ("Which is an input device?", ["Monitor","Printer","Keyboard","Speaker"], 2),
        ],
    },
    "Physics": {
        "Mechanics": [
            ("SI unit of force?", ["Newton","Joule","Watt","Pascal"], 0),
            ("Speed is distance divided by?", ["Mass","Time","Force","Energy"], 1),
        ],
        "General Physics": [
            ("SI unit of energy?", ["Newton","Joule","Watt","Volt"], 1),
            ("Approximate acceleration due to gravity on Earth?", ["3.8 m/s²","9.8 m/s²","15 m/s²","20 m/s²"], 1),
        ],
    },
    "Mathematics": {
        "Arithmetic": [
            ("What is 7 × 8?", ["54","56","64","48"], 1),
            ("What is 25 + 37?", ["52","62","72","57"], 1),
        ],
        "Algebra": [
            ("If x + 5 = 12, x = ?", ["5","6","7","8"], 2),
            ("What is the square root of 81?", ["7","8","9","10"], 2),
        ],
    },
}

def save_progress(subject, score, total):
    key = subject.replace(" ", "_")
    old = store.get(key) if store.exists(key) else {"attempts":0,"best":0,"answered":0}
    store.put(key, attempts=old["attempts"]+1,
              best=max(old["best"], score), answered=old["answered"]+total)

class Home(Screen):
    def on_enter(self):
        self.clear_widgets()
        box = BoxLayout(orientation="vertical", padding=dp(22), spacing=dp(12))
        box.add_widget(Label(text="[b]AL HADI[/b]", markup=True, font_size=34, size_hint_y=None, height=dp(65)))
        box.add_widget(Label(text="Learn • Practice • Improve", font_size=19, size_hint_y=None, height=dp(45)))
        for subject in QUESTIONS:
            b=Button(text=subject, font_size=20, size_hint_y=None, height=dp(60))
            b.bind(on_release=lambda btn,s=subject:self.open_subject(s))
            box.add_widget(b)
        p=Button(text="📊 My Progress", font_size=18, size_hint_y=None, height=dp(55))
        p.bind(on_release=lambda *_:setattr(self.manager,"current","progress"))
        box.add_widget(p)
        self.add_widget(box)

    def open_subject(self, subject):
        s=self.manager.get_screen("subject"); s.setup(subject); self.manager.current="subject"

class Subject(Screen):
    def setup(self, subject):
        self.subject=subject; self.clear_widgets()
        outer=BoxLayout(orientation="vertical",padding=dp(18),spacing=dp(10))
        outer.add_widget(Label(text=f"[b]{subject}[/b]",markup=True,font_size=28,size_hint_y=None,height=dp(55)))
        scroll=ScrollView()
        grid=GridLayout(cols=1,spacing=dp(10),padding=dp(5),size_hint_y=None)
        grid.bind(minimum_height=grid.setter("height"))
        for chapter in QUESTIONS[subject]:
            b=Button(text=f"📖 {chapter}",font_size=18,size_hint_y=None,height=dp(60))
            b.bind(on_release=lambda btn,c=chapter:self.start(c))
            grid.add_widget(b)
        scroll.add_widget(grid); outer.add_widget(scroll)
        back=Button(text="← Home",size_hint_y=None,height=dp(55))
        back.bind(on_release=lambda *_:setattr(self.manager,"current","home"))
        outer.add_widget(back); self.add_widget(outer)

    def start(self, chapter):
        q=self.manager.get_screen("quiz"); q.setup(self.subject,chapter); self.manager.current="quiz"

class Quiz(Screen):
    def setup(self,subject,chapter):
        self.subject,self.chapter=subject,chapter; self.items=QUESTIONS[subject][chapter]
        self.index=0; self.score=0; self.show()

    def show(self):
        self.clear_widgets()
        q,opts,correct=self.items[self.index]
        box=BoxLayout(orientation="vertical",padding=dp(18),spacing=dp(10))
        box.add_widget(Label(text=f"{self.subject} • {self.chapter}\nQuestion {self.index+1}/{len(self.items)}",font_size=20,size_hint_y=None,height=dp(75)))
        bar=ProgressBar(max=len(self.items),value=self.index,size_hint_y=None,height=dp(10)); box.add_widget(bar)
        box.add_widget(Label(text=q,font_size=23))
        for i,o in enumerate(opts):
            b=Button(text=o,font_size=18,size_hint_y=None,height=dp(58))
            b.bind(on_release=lambda btn,n=i:self.answer(n)); box.add_widget(b)
        self.add_widget(box)

    def answer(self,n):
        if n==self.items[self.index][2]: self.score+=1
        self.index+=1
        if self.index==len(self.items):
            save_progress(self.subject,self.score,len(self.items))
            r=self.manager.get_screen("result"); r.setup(self.subject,self.chapter,self.score,len(self.items)); self.manager.current="result"
        else: self.show()

class Result(Screen):
    def setup(self,subject,chapter,score,total):
        self.clear_widgets()
        box=BoxLayout(orientation="vertical",padding=dp(25),spacing=dp(18))
        box.add_widget(Label(text="[b]Quiz Complete![/b]",markup=True,font_size=30))
        box.add_widget(Label(text=f"{subject}\n{chapter}\n\nScore: {score}/{total}",font_size=24))
        again=Button(text="Try Again",size_hint_y=None,height=dp(60))
        again.bind(on_release=lambda *_:self.retry(subject,chapter)); box.add_widget(again)
        home=Button(text="Home",size_hint_y=None,height=dp(60))
        home.bind(on_release=lambda *_:setattr(self.manager,"current","home")); box.add_widget(home)
        self.add_widget(box)
    def retry(self,s,c):
        q=self.manager.get_screen("quiz"); q.setup(s,c); self.manager.current="quiz"

class Progress(Screen):
    def on_enter(self):
        self.clear_widgets()
        box=BoxLayout(orientation="vertical",padding=dp(20),spacing=dp(10))
        box.add_widget(Label(text="[b]My Progress[/b]",markup=True,font_size=30,size_hint_y=None,height=dp(60)))
        for subject in QUESTIONS:
            key=subject.replace(" ","_")
            d=store.get(key) if store.exists(key) else {"attempts":0,"best":0,"answered":0}
            box.add_widget(Label(text=f"{subject}\nAttempts: {d['attempts']}   Best: {d['best']}   Questions answered: {d['answered']}",font_size=17,size_hint_y=None,height=dp(80)))
        back=Button(text="← Home",size_hint_y=None,height=dp(58))
        back.bind(on_release=lambda *_:setattr(self.manager,"current","home")); box.add_widget(back)
        self.add_widget(box)

class AlHadiApp(App):
    def build(self):
        sm=ScreenManager()
        sm.add_widget(Home(name="home")); sm.add_widget(Subject(name="subject"))
        sm.add_widget(Quiz(name="quiz")); sm.add_widget(Result(name="result"))
        sm.add_widget(Progress(name="progress"))
        return sm

if __name__=="__main__": AlHadiApp().run()