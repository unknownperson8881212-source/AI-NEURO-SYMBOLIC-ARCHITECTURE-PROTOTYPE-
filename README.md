
import json
import os
from collections import defaultdict, deque
from datetime import datetime

# -----------------------------
# CONFIGURATION
# -----------------------------
KNOWLEDGE_FILE = "logic_knowledge.json"

class LogicCore:
    def __init__(self):
        # Graph Structure: Subject -> Relation -> [Objects]
        # This is the "Logical Brain" of the system
        self.kg = defaultdict(lambda: defaultdict(list))
        self.load_knowledge()

    # ==========================================
    # 1. LEARNING (Acquiring Facts)
    # ==========================================
    def learn_fact(self, subj, relation, obj, confidence=1.0):
        """
        Function to learn a new logical fact.
        It checks for validity before adding to memory.
        """
        subj = subj.lower().strip()
        obj = obj.lower().strip()
        
        # 1. Self-Reference Check
        if subj == obj:
            return f"❌ Error: '{subj}' cannot depend on itself (Self-loop).", False
        
        # 2. Cycle Detection (Circular Dependency Check)
        # If A needs B, and B needs A, this creates an infinite loop.
        if self._detect_cycle(subj, obj, relation):
             return f"❌ Critical Logic Error: Learning '{subj} -> {obj}' creates an infinite loop!", False

        # 3. Duplicate Check
        exists = any(o == obj for o, _, _ in self.kg[subj][relation])
        if not exists:
            # Store as: (Object, Confidence, Timestamp)
            self.kg[subj][relation].append((obj, confidence, datetime.now().isoformat()))
            self.save_knowledge()
            return f"✅ Learned Logic: {subj} --({relation})--> {obj}", True
        
        return f"ℹ️ Already known: {subj} {relation} {obj}", False

    # ==========================================
    # 2. REASONING (Finding Solutions)
    # ==========================================
    def find_solution_path(self, goal, relation_type="requires", max_depth=20):
        """
        Backward Chaining Algorithm.
        Starts from the goal and looks backwards to find requirements.
        """
        goal = goal.lower().strip()
        
        # Queue: (CurrentNode, PathTaken, Depth)
        q = deque([(goal, [], 0)])
        visited = set()
        
        possible_plans = []
        
        while q:
            curr, path, depth = q.popleft()
            
            # Depth limit to prevent infinite recursion
            if depth > max_depth:
                continue
            
            # Prevent local loops in path
            if curr in path: 
                continue
                
            if curr in visited: 
                continue
            visited.add(curr)
            
            # Base Case: If the current item has no further requirements
            if curr not in self.kg or relation_type not in self.kg[curr]:
                full_path = path + [curr]
                # Reverse the path to get Start -> Goal
                possible_plans.append(full_path[::-1])
                continue
            
            # Search for dependencies
            found_dependency = False
            for req, conf, _ in self.kg[curr][relation_type]:
                found_dependency = True
                q.append((req, path + [curr], depth + 1))
            
            # If node exists but has no active requirements (Dead end but valid root)
            if not found_dependency:
                full_path = path + [curr]
                possible_plans.append(full_path[::-1])

        if not possible_plans:
            return None, "No logical path found."
        
        # Select the shortest and most direct plan
        best_plan = min(possible_plans, key=len)
        return best_plan, "Success"

    # ==========================================
    # 3. SAFETY CHECKS
    # ==========================================
    def _detect_cycle(self, start_node, target_node, relation):
        """
        Checks if creating a new connection will cause an infinite loop.
        Uses DFS (Depth First Search).
        """
        # We are checking: Is there already a path from target_node back to start_node?
        # If yes, adding start_node -> target_node is dangerous.
        
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
    # 4. MEMORY PERSISTENCE
    # ==========================================
    def save_knowledge(self):
        try:
            # Convert defaultdict to normal dict for JSON serialization
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

# ==========================================
# EXAMPLE RUNNER (TESTING)
# ==========================================
if __name__ == "__main__":
    core = LogicCore()
    
    print("\n--- 1. TEACHING LOGIC ---")
    print(core.learn_fact("tea", "requires", "water")[0])
    print(core.learn_fact("water", "requires", "rain")[0])
    print(core.learn_fact("rain", "requires", "clouds")[0])
    print(core.learn_fact("tea", "requires", "milk")[0])
    
    print("\n--- 2. TESTING SAFETY (Cycle Detection) ---")
    # Invalid Logic Example: If clouds need tea, it creates a loop (Cloud -> Tea -> Water -> Rain -> Cloud)
    print(core.learn_fact("clouds", "requires", "tea")[0]) 
    
    print("\n--- 3. PLANNING (Solving Problems) ---")
    plan, status = core.find_solution_path("tea")
    
    if plan:
        print(f"✅ Logical Plan to make 'Tea':")
        print(" -> ".join(plan))
    else:
        print(f"❌ Plan failed: {status}")
