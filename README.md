
# ==========================================
# 1. LEARNING (तथ्य सीखना)
# ==========================================
def learn_fact(self, subj, relation, obj, confidence=1.0):
    """
    नया लॉजिक सीखने का फंक्शन।
    यह पहले चेक करता है कि कहीं यह लॉजिक असंभव तो नहीं है।
    """
    subj = subj.lower().strip()
    obj = obj.lower().strip()
    
    # 1. Self-Reference Check (खुद पर निर्भरता रोकना)
    if subj == obj:
        return f"❌ Error: '{subj}' cannot depend on itself (Self-loop).", False
    
    # 2. Cycle Detection (Circular Dependency Check)
    # अगर A को B चाहिए, और B को A चाहिए, तो यह सिस्टम को क्रैश कर देगा।
    if self._detect_cycle(subj, obj, relation):
         return f"❌ Critical Logic Error: Learning '{subj} -> {obj}' creates an infinite loop!", False

    # 3. Duplicate Check (डुप्लीकेट रोकना)
    exists = any(o == obj for o, _, _ in self.kg[subj][relation])
    if not exists:
        # (Object, Confidence, Timestamp)
        self.kg[subj][relation].append((obj, confidence, datetime.now().isoformat()))
        self.save_knowledge()
        return f"✅ Learned Logic: {subj} --({relation})--> {obj}", True
    
    return f"ℹ️ Already knew: {subj} {relation} {obj}", False

# ==========================================
# 2. REASONING (रास्ता और हल ढूँढना)
# ==========================================
def find_solution_path(self, goal, relation_type="requires", max_depth=20):
    """
    Backward Chaining Algorithm.
    यह लक्ष्य (Goal) से शुरू होकर पीछे की तरफ देखता है कि क्या-क्या चाहिए।
    """
    goal = goal.lower().strip()
    
    # Queue: (CurrentNode, PathTaken, Depth)
    q = deque([(goal, [], 0)])
    visited = set()
    
    possible_plans = []
    
    while q:
        curr, path, depth = q.popleft()
        
        if depth > max_depth:
            continue
        
        if curr in path: # लूप से बचाव
            continue
            
        if curr in visited: 
            continue
        visited.add(curr)
        
        # बेस केस: अगर इसके लिए कुछ नहीं चाहिए, तो हम बेस पर पहुंच गए
        if curr not in self.kg or relation_type not in self.kg[curr]:
            full_path = path + [curr]
            # प्लान को सीधा करें (Start -> Goal)
            possible_plans.append(full_path[::-1])
            continue
        
        # निर्भरता (Dependencies) ढूंढे
        found_dependency = False
        for req, conf, _ in self.kg[curr][relation_type]:
            found_dependency = True
            q.append((req, path + [curr], depth + 1))
        
        # अगर नोड है लेकिन कोई requirement नहीं है (Dead end but valid root)
        if not found_dependency:
            full_path = path + [curr]
            possible_plans.append(full_path[::-1])

    if not possible_plans:
        return None, "No logical path found."
    
    # सबसे छोटा और सटीक रास्ता चुनें
    best_plan = min(possible_plans, key=len)
    return best_plan, "Success"

# ==========================================
# 3. SAFETY CHECKS (सुरक्षा चक्र)
# ==========================================
def _detect_cycle(self, start_node, target_node, relation):
    """
    यह चेक करता है कि क्या नया कनेक्शन बनाने से 'गोल-गोल घूमने' वाली समस्या होगी।
    DFS (Depth First Search) का उपयोग।
    """
    # हम चेक कर रहे हैं: क्या target_node से वापस start_node तक आने का कोई रास्ता है?
    # अगर हाँ, तो start_node -> target_node जोड़ना गलत होगा।
    
    stack = [(target_node, [target_node])]
    visited = set()
    
    while stack:
        curr, path = stack.pop()
        if curr == start_node:
            return True # Cycle detected!
        
        if curr in visited: continue
        visited.add(curr)
        
        if curr in self.kg and relation in self.kg[curr]:
            for neighbor, _, _ in self.kg[curr][relation]:
                stack.append((neighbor, path + [neighbor]))
    
    return False

# ==========================================
# 4. MEMORY PERSISTENCE (याद रखना)
# ==========================================
def save_knowledge(self):
    try:
        # defaultdict को नार्मल dict में बदलना पड़ता है सेव करने के लिए
        data = {k: dict(v) for k, v in self.kg.items()}
        with open(KNOWLEDGE_FILE, 'w') as f:
            json.dump(data, f, indent=2)
    except Exception as e:
        print(f"⚠️ Save Error: {e}")

def load_knowledge(self):
    if os.path.exists(KNOWLEDGE_FILE):
        try:
            with open(KNOWLEDGE_FILE, 'r') as f:
                data = json.load(f)
                for subj, rels in data.items():
                    for r, objs in rels.items():
                        self.kg[subj][r] = objs
            print(f"🧠 Logic Core Loaded: {len(self.kg)} concepts ready.")
        except Exception as e:
            print(f"⚠️ Load Error: {e}")

def show_all_logic(self):
    output = []
    for subj, rels in self.kg.items():
        for r, objs in rels.items():
            for o, _, _ in objs:
                output.append(f"{subj} --{r}--> {o}")
    return "\n".join(output)
print("\n--- 1. TEACHING LOGIC ---")
print(brain.learn_fact("chai", "requires", "water")[0])
print(brain.learn_fact("water", "requires", "rain")[0])
print(brain.learn_fact("rain", "requires", "clouds")[0])
print(brain.learn_fact("chai", "requires", "milk")[0])

print("\n--- 2. TESTING SAFETY (Cycle Detection) ---")
# यह गलत लॉजिक है: अगर बादल को चाय चाहिए, तो साइकिल बन जाएगी (Cloud -> Chai -> Water -> Rain -> Cloud)
print(brain.learn_fact("clouds", "requires", "chai")[0]) 

print("\n--- 3. PLANNING (Solving Problems) ---")
plan, status = brain.find_solution_path("chai")

if plan:
    print(f"✅ Logical Plan to make 'Chai':")
    print(" -> ".join(plan))
else:
    print(f"❌ Plan failed: {status}")
