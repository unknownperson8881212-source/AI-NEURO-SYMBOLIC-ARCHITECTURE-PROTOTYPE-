def get_embedding(self, text):
    if not text.strip(): return torch.zeros(1, self.embed_dim).to(DEVICE)
    return torch.tensor(self.model.encode(text), device=DEVICE).unsqueeze(0)

def find_closest_semantic(self, query, candidates):
    if not candidates: return None
    
    query_emb = self.get_embedding(query)
    # Using cat to ensure correct tensor shape for comparison
    cand_embs = torch.cat([self.get_embedding(c) for c in candidates], dim=0)
    
    scores = F.cosine_similarity(query_emb, cand_embs)
    best_idx = torch.argmax(scores).item()
    
    if scores[best_idx].item() > 0.5: 
        return candidates[best_idx]
    return None
def forward(self, x):
    h = torch.zeros(1, self.hidden_dim, device=x.device)
    # Simulating time-steps for dynamic processing
    for _ in range(5): 
        f = torch.tanh(self.W_in(x) + self.W_rec(h))
        tau_eff = torch.sigmoid(self.tau).clamp(min=1e-2)
        h = h + (f - h) / tau_eff
    return F.softmax(self.classifier(h), dim=1)
def find_path(self, start_goal):
    # Backward Chaining Algorithm for Planning
    q = deque([(start_goal, [])])
    visited = set()
    
    while q:
        curr, path = q.popleft()
        if curr in visited: continue
        visited.add(curr)
        
        # Base case: If no requirements, we found the root
        if curr not in self.kg or "requires" not in self.kg[curr]:
            final_path = path + [curr]
            return final_path[::-1] # Return sequence: Start -> End
        
        # Recursive search for dependencies
        for req, _, _ in self.kg[curr]["requires"]:
            q.append((req, path + [curr]))
            
    return None

def learn_fact(self, subj, relation, obj):
    # Avoid duplicates
    exists = any(o == obj for o, _, _ in self.kg[subj][relation])
    if not exists:
        self.kg[subj][relation].append((obj, 1.0, datetime.now().isoformat()))
        return f"Learned: {subj} -> {relation} -> {obj}"
    return f"I already know that {subj} {relation} {obj}"
def process_input(self, user_text):
    user_text = user_text.lower().strip()
    
    # --- CAPABILITY 1: PLANNING ---
    if user_text.startswith("plan"):
        goal = user_text.replace("plan", "").strip()
        matched_goal = self.perception.find_closest_semantic(goal, list(self.logic.kg.keys()))
        
        if matched_goal:
            path = self.logic.find_path(matched_goal)
            if path: return f"LOGICAL PLAN: {' -> '.join(path)}"
            else: return f"Planning Error: Missing requirements for '{matched_goal}'."
        else:
            return "Goal unknown. Please teach me the requirements first."

    # --- CAPABILITY 2: LEARNING (Parsing) ---
    relation = None
    splitter = None
    
    if "needs" in user_text: 
        relation = "requires"; splitter = "needs"
    elif "requires" in user_text: 
        relation = "requires"; splitter = "requires"
    elif "is a" in user_text: 
        relation = "is_a"; splitter = "is a"
    
    if relation and splitter:
        try:
            parts = user_text.split(splitter)
            if len(parts) == 2:
                subj = parts[0].strip()
                obj = parts[1].strip()
                if subj and obj:
                    return self.logic.learn_fact(subj, relation, obj)
        except:
            pass

    return "System listening. Teach me facts (e.g., 'Fire requires Wood') or ask for a Plan."
print("\n👇 Test Scenario:")
print("1. Teach: 'tea needs water'")
print("2. Teach: 'water needs rain'")
print("3. Command: 'plan tea'")

while True:
    try:
        txt = input("\nUser: ")
        if not txt: continue
        if txt.lower() in ["exit", "quit"]: break
        
        print(f"🤖 Agent: {agent.process_input(txt)}")
        
    except Exception as e:
        print(f"❌ System Error: {e}")
