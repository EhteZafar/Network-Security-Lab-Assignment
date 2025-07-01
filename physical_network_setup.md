# Physical Network Setup Guide - Step by Step
==============================================

## 🎯 **Goal**
Set up a secure 3-zone network with proper physical connections for the Network Security Lab.

## 📦 **Equipment Checklist**
Before starting, make sure you have:
- [ ] 5 grml computers (labeled grml1 through grml5)
- [ ] 2-3 Ethernet switches (small boxes with multiple network ports)
- [ ] 8-10 Ethernet cables (patch cables with RJ45 connectors)
- [ ] Access to wall port EV3-2-2 5/9 or EV3-2-2 5/21

---

## 🏗️ **Network Architecture Overview**

```
🌐 INTERNET (141.76.46.0/24)
     ↓
  Wall Port EV3-2-2
     ↓
┌─────────────────┐
│     grml1       │ ← Main Traffic Controller (NAT Gateway)
│  (3 interfaces) │
└─────┬─────┬─────┘
      ↓     ↓
   DMZ     INTERNAL
   Zone     Zone
     ↓       ↓
  grml3   grml2,grml4,grml5
 (Web)    (Clients/DNS)
```

---

## 🔌 **Step 1: Connect grml1 (NAT Gateway) - The Central Hub**

**grml1 is your network's "traffic director" and needs 3 separate connections:**

### 🌐 Connection A: Internet Access (eth0)
```
Internet → Wall Port → Cable → grml1 eth0
```
**Instructions:**
1. Locate wall port **EV3-2-2 5/9** or **EV3-2-2 5/21**
2. Take 1 Ethernet cable
3. Plug one end into the **wall port**
4. Plug other end into **grml1's eth0** (usually first/leftmost Ethernet port)
5. **Result:** grml1 now has Internet access (141.76.46.220)

### 🔒 Connection B: DMZ Network (eth1)
```
grml1 eth1 → Cable → Switch A
```
**Instructions:**
1. Take 1 Ethernet cable
2. Plug one end into **grml1's eth1** (second Ethernet port)
3. Plug other end into **Switch A** (any port - we'll call this "DMZ Switch")
4. **Label Switch A** as "DMZ" to avoid confusion
5. **Result:** grml1 can now control DMZ traffic (192.168.2.1)

### 🏠 Connection C: Internal Network (eth2)
```
grml1 eth2 → Cable → Switch B
```
**Instructions:**
1. Take 1 Ethernet cable
2. Plug one end into **grml1's eth2** (third Ethernet port)
3. Plug other end into **Switch B** (any port - we'll call this "Internal Switch")
4. **Label Switch B** as "INTERNAL" to avoid confusion
5. **Result:** grml1 can now control internal traffic (192.168.1.1)

---

## 🔒 **Step 2: Connect DMZ Zone (Semi-Public Area)**

**DMZ = Demilitarized Zone - for your web server**

### Connect grml3 (Web Server)
```
Switch A (DMZ) → Cable → grml3 eth0
```
**Instructions:**
1. Take 1 Ethernet cable
2. Plug one end into **Switch A** (any free port)
3. Plug other end into **grml3's eth0**
4. **Result:** grml3 is now in the DMZ (will be 192.168.2.10)

**DMZ Zone Complete:**
- Switch A has 2 cables: grml1 eth1 + grml3
- grml3 is isolated from internal machines
- grml3 can only reach Internet and DNS (controlled by grml1)

---

## 🏠 **Step 3: Connect Internal Zone (Private Area)**

**Internal = Your secure private network**

### Connect All Internal Machines
```
Switch B (Internal) connections:
├─ grml1 eth2 (already connected)
├─ grml2 (DNS Server)
├─ grml4 (Client 1)
└─ grml5 (Client 2)
```

**Instructions:**
1. **grml2 (DNS):** Cable from Switch B → grml2 eth0
2. **grml4 (Client 1):** Cable from Switch B → grml4 eth0
3. **grml5 (Client 2):** Cable from Switch B → grml5 eth0

**Internal Zone Complete:**
- Switch B has 4 cables total
- All internal machines can talk to each other
- All can reach Internet through grml1

---

## 📊 **Final Physical Layout**

```
                    🌐 INTERNET
                         ↓
                   Wall Port EV3-2-2
                         ↓
                    ┌─────────┐
                    │  grml1  │
                    │   NAT   │
                    └─┬─┬─┬───┘
                    eth0│ │eth2
                        │ │
                     eth1│ └─────┐
                         │       │
               ┌─────────▼─┐   ┌─▼─────────────────┐
               │ Switch A  │   │    Switch B       │
               │   (DMZ)   │   │  (INTERNAL)       │
               └─────┬─────┘   └─┬─┬─┬─────────────┘
                     │           │ │ │
                ┌────▼────┐     │ │ └───grml5 (CLT2)
                │  grml3  │     │ └─────grml4 (CLT1)
                │  (WEB)  │     └───────grml2 (DNS)
                └─────────┘
```

---

## 🔍 **Verification Checklist**

After connecting all cables, verify:

### Visual Check
- [ ] All Ethernet ports show **green/orange LED lights**
- [ ] **7 total cables** used (1 to wall + 6 between machines/switches)
- [ ] **grml1 has 3 cables** (eth0, eth1, eth2)
- [ ] **Switch A has 2 cables** (grml1 + grml3)
- [ ] **Switch B has 4 cables** (grml1 + 3 internal machines)

### Physical Connection Map
```
Cable 1: Wall Port EV3-2-2 → grml1 eth0
Cable 2: grml1 eth1 → Switch A
Cable 3: grml1 eth2 → Switch B
Cable 4: Switch A → grml3 eth0
Cable 5: Switch B → grml2 eth0
Cable 6: Switch B → grml4 eth0
Cable 7: Switch B → grml5 eth0
```

### Zone Verification
- [ ] **DMZ Zone:** Only grml3 connected
- [ ] **Internal Zone:** grml2, grml4, grml5 connected
- [ ] **No cross-connections:** grml3 NOT on internal switch

---

## 🚨 **Common Mistakes & Troubleshooting**

### Mistake 1: Wrong grml1 Interfaces
**Problem:** Mixed up eth0, eth1, eth2 on grml1
**Solution:** 
- eth0 = Internet (to wall)
- eth1 = DMZ (to Switch A)
- eth2 = Internal (to Switch B)

### Mistake 2: Mixed Network Zones
**Problem:** grml3 connected to internal switch
**Solution:** grml3 MUST be on Switch A (DMZ only)

### Mistake 3: No LED Lights
**Problem:** Ethernet port shows no lights
**Solution:** 
- Check cable is fully inserted (should "click")
- Try different cable
- Try different switch port

### Mistake 4: Switch Confusion
**Problem:** Don't know which switch is which
**Solution:** 
- Label switches clearly: "DMZ" and "INTERNAL"
- Count connections: DMZ=2 cables, Internal=4 cables

---

## 🎯 **What Happens Next**

After physical setup is complete:
1. **Configure IP addresses** on each machine
2. **Set up iptables rules** on grml1 for security
3. **Install services** (DNS on grml2, Web on grml3)
4. **Test connectivity** between zones

**Remember:** Physical connections create the foundation, but software configuration controls the security!

---

## 📞 **Quick Reference**

### IP Address Plan
- **grml1:** 141.76.46.220 (eth0), 192.168.2.1 (eth1), 192.168.1.1 (eth2)
- **grml2:** 192.168.1.2 (DNS Server)
- **grml3:** 192.168.2.10 (Web Server - DMZ)
- **grml4:** 192.168.1.4 (Client 1)
- **grml5:** 192.168.1.5 (Client 2)

### Network Purposes
- **External (141.76.46.x):** Internet connectivity
- **DMZ (192.168.2.x):** Public web services (isolated)
- **Internal (192.168.1.x):** Private resources (protected)

**✅ Physical setup complete! Ready for software configuration.** 