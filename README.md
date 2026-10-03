// =====================================================================
// Sandboxels mod: WARRIORS + VEHICLES  (everything in one file)
//   Warriors: teams, swords, spears, shields, armor, guns, healing potion,
//             missile launcher, peace/war button.
//   Vehicles: tank, helicopter, plane (use the same Red/Blue Team tools).
// Paste into the Console (F12) after the game loads.
// =====================================================================

// Sandboxels mod: Warriors v2
// Neutral warriors that mine + craft. Use the team tools to assign Red/Blue,
// then use the PEACE/WAR button to let the teams fight.

(function () {
  const VIEW = 5;
  const SCAN_EVERY = 10;
  const MAX_HP = 20;
  const POTION_HEAL = 0.2;   // fraction of max health restored
  const MISSILE_RADIUS = 10;   // explosion radius in pixels (about two missiles' blasts side by side)
  const MISSILE_DAMAGE = 100;  // anything caught in the blast dies (ignores armor and shields)
  const MISSILE_SPEED = 2;     // cells per tick
  const SMALL_ARMS_VS_VEHICLE = 0.5;   // bullets, spears and swords only do this fraction to tanks/helis
  const MISSILE_VEHICLE_DAMAGE = 60;   // launcher explosion damage to a vehicle (people still take 100)
  const MISSILE_RISE = 30;     // how high it climbs before turning toward the enemy
  const LAUNCH_CD = 150;       // ticks between launches
  const LAUNCH_MIN = 14;       // won't fire at targets closer than this (avoids self-blast)
  const LAUNCH_MAX = 90;
  const LAUNCHER_RGB = { none: [110, 110, 120], red: [190, 70, 60], blue: [60, 110, 190] };
  const MAX_GEAR = 3;
  const MAX_SHIELD = 2;
  const MAX_SPEARS = 3;
  const BULLET_SPEED = 2;   // cells per tick
  // Guns: infinite ammo. range/keep are in pixels; pellets = rows fired (offset from chest)
  // Gun ranking: pistol < shotgun = rifle. A warrior only swaps to a BETTER tier (and drops the old gun).
  const GUNS = {
    pistol:  { tier: 1, label: "Pistol",  range: 12, dmg: 3, cd: 14, keep: 3, pellets: [0],     rgb: [70, 70, 80] },
    shotgun: { tier: 2, label: "Shotgun", range: 8,  dmg: 9, cd: 28, keep: 2, pellets: [-1, 0], rgb: [120, 75, 40] },
    rifle:   { tier: 2, label: "Rifle",   range: 60, dmg: 5, cd: 24, keep: 10, pellets: [0],     rgb: [45, 70, 55] },
  };
  const MAX_FIGHTERS = 3;   // per team at the front; the rest support
  const BASE = {
    none: [190, 165, 125],
    red:  [200, 60, 50],
    blue: [50, 110, 200],
  };
  let warMode = false;
  // Bridge so warriors and vehicles (same file, below) can talk to each other
  const API = window.__warVehicleAPI = {};
  const missileTargets = new Map();                   // launcher missile id -> its selected target
  API.isWar = function () { return warMode; };
  API.setTeam = function (p, t) { setTeam(p, t); };

  // Roster of teamed warriors, rebuilt every frame, so warriors can find the
  // nearest enemy anywhere on the map (stored in the mod, NOT on pixels).
  let rosterTick = -1;
  let rosterCur = { red: [], blue: [] };
  let rosterPrev = { red: [], blue: [] };
  let fightPrev = { red: 0, blue: 0 };
  let fightCur = { red: 0, blue: 0 };
  let promoted = { red: 0, blue: 0 };
  function register(pixel) {
    if (!pixel.team || !rosterCur[pixel.team]) return;
    if (rosterTick !== pixelTicks) {
      rosterPrev = rosterCur;
      rosterCur = { red: [], blue: [] };
      fightPrev = fightCur;
      fightCur = { red: 0, blue: 0 };
      promoted = { red: 0, blue: 0 };
      rosterTick = pixelTicks;
    }
    rosterCur[pixel.team].push(pixel);
    if (pixel.fighter) fightCur[pixel.team]++;
  }

  // A warrior joins the front line only if its team has fewer than 3 fighters
  function tryPromote(pixel) {
    if (fightPrev[pixel.team] + promoted[pixel.team] >= MAX_FIGHTERS) return false;
    pixel.fighter = true;
    promoted[pixel.team]++;
    return true;
  }

  function clearBlood(x, y) {
    if (outOfBounds(x, y)) return;
    const p = pixelMap[x][y];
    if (p && p.element === "blood") deletePixel(x, y);
  }

  function isWarrior(p) { return p && p.element === "warrior"; }
  function isWarriorPart(p) {
    return p && (p.element === "warrior" || p.element === "warrior_head");
  }
  function isEnemy(a, b) {
    return isWarrior(b) && a.team && b.team && a.team !== b.team;
  }

  function tint(pixel) {
    const b = BASE[pixel.team || "none"];
    const g = pixel.armor * 25;
    const s = pixel.weapon * 20;
    const c = [b[0] + s + g, b[1] + s + g, b[2] + s + g]
      .map(v => Math.max(0, Math.min(255, Math.round(v))));
    pixel.color = "rgb(" + c.join(",") + ")";
    // head: skin normally, steel-grey helmet once armored
    const h = getHead(pixel);
    if (h) {
      const a = 150 + pixel.armor * 25;
      h.color = pixel.armor > 0 ? "rgb(" + a + "," + a + "," + (a + 10) + ")" : "rgb(232,185,138)";
    }
  }

  function getHead(pixel) {
    if (outOfBounds(pixel.x, pixel.y - 1)) return null;
    const h = pixelMap[pixel.x][pixel.y - 1];
    return (h && h.element === "warrior_head") ? h : null;
  }

  function ensureHead(pixel) {
    if (getHead(pixel)) return;
    if (isEmpty(pixel.x, pixel.y - 1)) {
      createPixel("warrior_head", pixel.x, pixel.y - 1);
      tint(pixel);
    }
  }

  function fall(pixel) {
    const head = getHead(pixel);
    if (tryMove(pixel, pixel.x, pixel.y + 1)) {
      if (head) tryMove(head, pixel.x, pixel.y - 1);
      return true;
    }
    return false;
  }

  function headTick(head) {
    const below = outOfBounds(head.x, head.y + 1) ? null : pixelMap[head.x][head.y + 1];
    if (isWarrior(below)) { head.orph = 0; return; }   // attached to a body
    // orphaned head (body died some other way): turns into blood
    head.orph = (head.orph || 0) + 1;
    if (head.orph > 3) changePixel(head, "blood");
  }

  function init(pixel) {
    pixel.hp = MAX_HP;
    pixel.weapon = 0;
    pixel.armor = 0;
    pixel.iron = 0;
    pixel.wood = 0;
    pixel.cd = 0;
    pixel.spear = 0;
    pixel.shield = 0;
    pixel.throwCd = 0;
    pixel.flash = 0;
    pixel.dir = Math.random() < 0.5 ? -1 : 1;
    pixel.ready = true;
  }

  function craft(pixel) {
    // Sword: 1 wood + 1 iron.  Armor: 3 iron.
    // Shield: 2 wood + 1 iron.  Spear: 2 wood.
    if (pixel.weapon < MAX_GEAR && pixel.wood >= 1 && pixel.iron >= 1) {
      pixel.wood--; pixel.iron--; pixel.weapon++; tint(pixel);
    } else if (pixel.armor < MAX_GEAR && pixel.iron >= 3) {
      pixel.iron -= 3; pixel.armor++; tint(pixel);
    } else if (pixel.shield < MAX_SHIELD && pixel.wood >= 2 && pixel.iron >= 1) {
      pixel.wood -= 2; pixel.iron--; pixel.shield++;
    } else if (pixel.spear < MAX_SPEARS && pixel.wood >= 2) {
      pixel.wood -= 2; pixel.spear++;
    } else if (!pixel.gun && pixel.iron >= 4) {
      pixel.iron -= 4; pixel.gun = "pistol";
    }
  }

  // Put an item on the ground next to a warrior (used when it swaps to a better gun)
  function dropItem(element, pixel) {
    const spots = [[-1, 0], [1, 0], [-2, 0], [2, 0], [0, -3], [-1, -1], [1, -1]];
    for (const o of spots) {
      const x = pixel.x + o[0], y = pixel.y + o[1];
      if (isEmpty(x, y)) { createPixel(element, x, y); return true; }
    }
    return false;
  }

  function pickup(pixel) {
    for (let dx = -2; dx <= 2; dx++) {
      for (let dy = -2; dy <= 3; dy++) {          // dy > 0 = below the warrior
        const x = pixel.x + dx, y = pixel.y + dy;
        if (outOfBounds(x, y)) continue;
        const p = pixelMap[x][y];
        if (!p) continue;
        if (p.element === "warrior_sword" && pixel.weapon < MAX_GEAR) {
          deletePixel(x, y); pixel.weapon++; tint(pixel); return true;
        }
        if (p.element === "warrior_armor" && pixel.armor < MAX_GEAR) {
          deletePixel(x, y); pixel.armor++; tint(pixel); return true;
        }
        if (p.element === "warrior_potion" && pixel.hp < MAX_HP) {
          deletePixel(x, y);
          pixel.hp = Math.min(MAX_HP, pixel.hp + MAX_HP * POTION_HEAL);
          flashWarrior(pixel, "rgb(90,255,120)");
          return true;
        }
        for (const k in GUNS) {
          const cur = GUNS[pixel.gun] ? GUNS[pixel.gun].tier : 0;
          if (p.element === "warrior_" + k && cur < GUNS[k].tier) {
            deletePixel(x, y);
            if (pixel.gun) dropItem("warrior_" + pixel.gun, pixel);   // leave the worse gun for someone else
            pixel.gun = k;
            return true;
          }
        }
        if (p.element === "warrior_spear" && pixel.spear < MAX_SPEARS) {
          deletePixel(x, y); pixel.spear++; return true;
        }
        if (p.element === "warrior_shield" && pixel.shield < MAX_SHIELD) {
          deletePixel(x, y); pixel.shield++; return true;
        }
      }
    }
    return false;
  }

  function mine(pixel) {
    for (let dx = -1; dx <= 1; dx++) {
      for (let dy = -1; dy <= 1; dy++) {
        const x = pixel.x + dx, y = pixel.y + dy;
        if (outOfBounds(x, y)) continue;
        const p = pixelMap[x][y];
        if (!p) continue;
        if (p.element === "iron" || p.element === "steel") {
          deletePixel(x, y); pixel.iron++; craft(pixel); return true;
        }
        if (p.element === "wood") {
          deletePixel(x, y); pixel.wood++; craft(pixel); return true;
        }
      }
    }
    return false;
  }

  // ----- vehicles that warriors are allowed to attack: tanks and helicopters, NEVER planes -----
  function isAttackableVehicle(v) { return !!v && (v.element === "veh_tank" || v.element === "veh_heli"); }
  function vehicleAlive(v) { return isAttackableVehicle(v) && !outOfBounds(v.x, v.y) && pixelMap[v.x][v.y] === v; }
  function enemyVehicles(team) { return API.enemyVehicles ? API.enemyVehicles(team) : []; }   // enemy tanks/helis only
  function targetAlive(t) {
    if (isWarrior(t)) return !outOfBounds(t.x, t.y) && pixelMap[t.x][t.y] === t;
    return vehicleAlive(t);
  }
  // for a vehicle, aim at the part of it closest to us (it's many pixels wide)
  function aimPoint(pixel, tgt) {
    if (isAttackableVehicle(tgt) && API.nearestPart) return API.nearestPart(tgt, pixel.x, pixel.y) || tgt;
    return tgt;
  }

  function findTarget(pixel) {
    const list = rosterPrev[pixel.team === "red" ? "blue" : "red"] || [];
    let best = null, bestD = Infinity;
    for (const p of list) {
      if (!isWarrior(p) || outOfBounds(p.x, p.y) || pixelMap[p.x][p.y] !== p) continue;
      const d = Math.abs(p.x - pixel.x) + Math.abs(p.y - pixel.y);
      if (d < bestD) { bestD = d; best = p; }
    }
    for (const v of enemyVehicles(pixel.team)) {
      if (!vehicleAlive(v)) continue;
      const a2 = aimPoint(pixel, v);
      const d = Math.abs(a2.x - pixel.x) + Math.abs(a2.y - pixel.y);
      if (d < bestD) { bestD = d; best = v; }
    }
    return best;
  }

  function flashWarrior(w, color) {
    w.color = color; w.flash = 4;
    const h = getHead(w);
    if (h) h.color = color;
  }

  function knock(w, dir) {
    if (w.mount) return;                            // can't knock a rider off a horse
    const head = getHead(w);
    const nx = w.x + dir;
    if (!isEmpty(nx, w.y)) return;
    if (head && !isEmpty(nx, w.y - 1)) return;
    if (tryMove(w, nx, w.y) && head) tryMove(head, nx, w.y - 1);
  }

  // One place for all damage: shield block -> armor -> hp -> death/loot
  function hurt(target, dmg, attacker, knockDir, pierce) {
    if (!pierce && target.shield && Math.random() < 0.35 * target.shield) {
      flashWarrior(target, "rgb(120,200,255)");      // blocked!
      return false;
    }
    if (!pierce) dmg = Math.max(1, dmg - (target.armor || 0));
    target.hp -= dmg;
    if (target.hp <= 0) {
      if (attacker) {                                  // loot the better gear
        attacker.weapon = Math.max(attacker.weapon, target.weapon || 0);
        attacker.armor = Math.max(attacker.armor, target.armor || 0);
        attacker.shield = Math.max(attacker.shield || 0, target.shield || 0);
        attacker.iron += 1;
        craft(attacker);
        tint(attacker);
      }
      // no head left behind: remove it and leave just blood
      const dead = getHead(target);
      if (dead) deletePixel(dead.x, dead.y);
      changePixel(target, "blood");
      return true;
    }
    flashWarrior(target, "rgb(255,255,255)");          // hit flash
    if (knockDir) knock(target, knockDir);
    return false;
  }

  function attack(pixel, enemy, dx) {
    if (pixel.cd > 0) return;
    pixel.cd = 6;
    if (enemy.element && enemy.element.indexOf("veh_") === 0) {          // hitting a tank / helicopter
      API.damageVeh(enemy, Math.max(2 + pixel.weapon * 2, (pixel.spear > 0 || pixel.gun) ? 3 : 0) * SMALL_ARMS_VS_VEHICLE);
      return;
    }
    const dir = enemy.x >= pixel.x ? 1 : -1;
    const swordDmg = 2 + pixel.weapon * 2;
    if (dx <= 1) {
      // close: best of sword / spear thrust (3) / fists (2). Sword knocks back.
      const dmg = Math.max(swordDmg, (pixel.spear > 0 || pixel.gun) ? 3 : 0);
      hurt(enemy, dmg, pixel, pixel.weapon > 0 ? dir : 0);
    } else if (pixel.spear > 0) {
      hurt(enemy, 3, pixel, 0);                        // spear thrust, reach 2
    }
  }

  function bodyOf(t) {
    if (!t) return null;
    if (t.element === "warrior") return t;
    if (t.element === "warrior_head" && !outOfBounds(t.x, t.y + 1) && isWarrior(pixelMap[t.x][t.y + 1])) {
      return pixelMap[t.x][t.y + 1];
    }
    return null;
  }

  // Lob a spear in an arc: starts above the thrower's head, peaks, then
  // comes down on the target. Friends are passed over, enemies get hit.
  function throwSpear(pixel, tgt) {
    if (!(pixel.spear > 0) || (pixel.throwCd || 0) > 0) return;
    const dir = tgt.x >= pixel.x ? 1 : -1;
    pixel.dir = dir;
    const x = pixel.x + dir, y = pixel.y - 2;           // above heads
    if (!isEmpty(x, y)) return;
    createPixel("warrior_spear_proj", x, y);
    const pr = pixelMap[x][y];
    if (!pr) return;
    const D = Math.max(3, Math.abs(tgt.x - x));
    pr.dir = dir; pr.team = pixel.team;
    pr.x0 = x; pr.y0 = y; pr.T = D;                      // flight length
    pr.H = Math.max(3, Math.round(D / 3));               // arc height
    pr.dy = tgt.y - y;                                   // land on target's row
    pr.t = 0;
    pixel.throwCd = 20;                                  // spears are never used up
  }

  function killProj(p) {
    if (p.trx !== undefined && !outOfBounds(p.trx, p.try_)) {
      const tr = pixelMap[p.trx][p.try_];
      if (tr && tr.element === "warrior_spear_trail") deletePixel(p.trx, p.try_);
    }
    deletePixel(p.x, p.y);
  }

  function projTick(p) {
    if (p.t === undefined) return;
    if (pixelTicks % 2) return;                          // slow enough to watch
    p.t++;
    if (p.t > p.T * 1.6 + 4) { killProj(p); return; }
    const u = p.t / p.T;
    const nx = p.x0 + p.dir * p.t;
    let ny = Math.round(p.y0 - 4 * p.H * u * (1 - u) + p.dy * u);
    if (outOfBounds(nx, ny)) { killProj(p); return; }

    let hit = pixelMap[nx][ny];
    if (hit) {
      const body = bodyOf(hit);
      const pveh = API.vehicleOf ? API.vehicleOf(hit) : null;
      if (pveh && isAttackableVehicle(pveh) && pveh.team && pveh.team !== p.team) {   // enemy tank/heli
        API.damageVeh(pveh, 5 * SMALL_ARMS_VS_VEHICLE);
        killProj(p);
        return;
      }
      if (body && body.team && body.team !== p.team) {   // enemy: hit!
        hurt(body, 5, null, p.dir);
        killProj(p);
        return;
      }
      if (body || pveh) {                                // friend/neutral/plane: fly over
        ny -= 1;
        if (outOfBounds(nx, ny) || pixelMap[nx][ny]) ny -= 1;
        if (outOfBounds(nx, ny) || pixelMap[nx][ny]) { killProj(p); return; }
      } else {                                           // hit the ground / a wall
        killProj(p);
        return;
      }
    }

    const ox = p.x, oy = p.y;
    if (tryMove(p, nx, ny)) {
      // keep exactly one trailing piece behind the tip: the "--" look
      if (p.trx !== undefined && !outOfBounds(p.trx, p.try_)) {
        const old = pixelMap[p.trx][p.try_];
        if (old && old.element === "warrior_spear_trail") deletePixel(p.trx, p.try_);
      }
      if (isEmpty(ox, oy)) {
        createPixel("warrior_spear_trail", ox, oy);
        p.trx = ox; p.try_ = oy;
      }
    }
  }

  function shoot(pixel, tgt) {
    const g = GUNS[pixel.gun];
    if (!g || pixel.gunCd > 0) return;
    pixel.dir = tgt.x >= pixel.x ? 1 : -1;
    let fired = false;
    for (const off of g.pellets) {
      const x = pixel.x + pixel.dir, y = pixel.y + off;
      if (!isEmpty(x, y)) continue;
      createPixel("warrior_bullet", x, y);
      const bl = pixelMap[x][y];
      if (!bl) continue;
      const dx = tgt.x - x, dy = tgt.y - y, m = Math.hypot(dx, dy) || 1;
      bl.team = pixel.team; bl.dmg = g.dmg;
      bl.fx = x; bl.fy = y;                       // exact (float) position
      bl.ux = dx / m; bl.uy = dy / m;             // aim direction
      bl.life = Math.ceil(g.range / BULLET_SPEED) + 4;
      fired = true;
    }
    if (fired) pixel.gunCd = g.cd;
  }

  // AIMBOT bullet: flies at any angle and keeps steering toward the nearest
  // enemy every tick. Friends and blood are passed through; walls stop it.
  function bulletTick(bl) {
    if (bl.life === undefined) return;
    bl.life--;
    if (bl.life < 0) { deletePixel(bl.x, bl.y); return; }

    const list = rosterPrev[bl.team === "red" ? "blue" : "red"] || [];
    let tx = null, ty = null, bd = Infinity;
    for (const w of list) {
      if (!isWarrior(w) || outOfBounds(w.x, w.y) || pixelMap[w.x][w.y] !== w) continue;
      const d = Math.abs(w.x - bl.fx) + Math.abs(w.y - bl.fy);
      if (d < bd) { bd = d; tx = w.x; ty = w.y; }
    }
    for (const v of enemyVehicles(bl.team)) {                  // enemy tanks / helicopters count as targets
      if (!vehicleAlive(v)) continue;
      const pt = API.nearestPart ? (API.nearestPart(v, Math.round(bl.fx), Math.round(bl.fy)) || v) : v;
      const d = Math.abs(pt.x - bl.fx) + Math.abs(pt.y - bl.fy);
      if (d < bd) { bd = d; tx = pt.x; ty = pt.y; }
    }
    if (tx !== null) {
      const dx = tx - bl.fx, dy = ty - bl.fy, m = Math.hypot(dx, dy) || 1;
      bl.ux = dx / m; bl.uy = dy / m;
    }

    let lx = bl.x, ly = bl.y;
    for (let i = 0; i < BULLET_SPEED; i++) {
      bl.fx += bl.ux; bl.fy += bl.uy;
      const cx = Math.round(bl.fx), cy = Math.round(bl.fy);
      if (cx === lx && cy === ly) continue;
      if (outOfBounds(cx, cy)) { deletePixel(bl.x, bl.y); return; }
      const t = pixelMap[cx][cy];
      if (t && t !== bl) {
        const body = bodyOf(t);
        if (body && body.team && body.team !== bl.team) {    // enemy: hit
          hurt(body, bl.dmg, null, bl.ux >= 0 ? 1 : -1);
          deletePixel(bl.x, bl.y);
          return;
        }
        const veh = API.vehicleOf ? API.vehicleOf(t) : null;
        if (veh) {
          if (veh.element === "veh_plane") continue;           // warriors never attack planes
          if (veh.team && veh.team === bl.team) continue;      // friendly vehicle: pass over it
          API.damageVeh(veh, bl.dmg * SMALL_ARMS_VS_VEHICLE);
          deletePixel(bl.x, bl.y);
          return;
        }
        if (body || t.element === "blood") continue;          // pass through
        deletePixel(bl.x, bl.y);                              // wall
        return;
      }
      lx = cx; ly = cy;
    }
    if (lx !== bl.x || ly !== bl.y) {
      const ox = bl.x, oy = bl.y;
      tryMove(bl, lx, ly);                                  // plain bullet: no flame trail
    }
  }

  // ---------- Missile launcher ----------
  function paintLauncher(L) {
    const c = LAUNCHER_RGB[L.team || "none"];
    L.color = "rgb(" + c.join(",") + ")";
  }

  function explosion(x, y, r) {
    const bodies = new Map();
    const vehs = new Set();
    for (let dx = -r; dx <= r; dx++) {
      for (let dy = -r; dy <= r; dy++) {
        const d = Math.hypot(dx, dy);
        if (d > r) continue;
        const cx = x + dx, cy = y + dy;
        if (outOfBounds(cx, cy)) continue;
        const p = pixelMap[cx][cy];
        if (p) {
          const body = bodyOf(p);
          if (body) { bodies.set(body, Math.hypot(body.x - x, body.y - y)); continue; }
          const veh = API.vehicleOf ? API.vehicleOf(p) : null;
          if (veh) { if (veh.element !== "veh_plane") vehs.add(veh); continue; }
          const info = elements[p.element];
          if (info && (info.hardness || 0) >= 0.9) continue;     // walls survive
          if (Math.random() < 1 - 0.6 * (d / r)) deletePixel(cx, cy);
        } else if (d > r * 0.4 && Math.random() < 0.12) {
          createPixel("fire", cx, cy);                           // scattered flames
        }
      }
    }
    for (const veh of vehs) API.damageVeh(veh, MISSILE_VEHICLE_DAMAGE);
    for (const [body, dist] of bodies) {
      if (outOfBounds(body.x, body.y) || pixelMap[body.x][body.y] !== body) continue;
      hurt(body, MISSILE_DAMAGE, null, body.x >= x ? 1 : -1, true);   // instant kill
    }
  }

  // Flight path: straight UP -> right angle toward the enemy -> straight DOWN
  function missileTick(m) {
    if (m.life === undefined) return;
    m.age = (m.age || 0) + 1;
    if (--m.life < 0) { missileTargets.delete(m.mid); const x = m.x, y = m.y; deletePixel(x, y); explosion(x, y, MISSILE_RADIUS); return; }

    for (let i = 0; i < MISSILE_SPEED; i++) {
      if (m.phase === 1) {                       // stay locked on the SELECTED target
        let sel = missileTargets.get(m.mid);
        if (targetAlive(sel)) {
          m.tx = sel.x;
        } else {                                 // target is gone: lock onto the nearest enemy
          const list = rosterPrev[m.team === "red" ? "blue" : "red"] || [];
          let bd = Infinity;
          for (const w of list) {
            if (!isWarrior(w) || outOfBounds(w.x, w.y) || pixelMap[w.x][w.y] !== w) continue;
            const d = Math.abs(w.x - m.x);
            if (d < bd) { bd = d; m.tx = w.x; missileTargets.set(m.mid, w); }
          }
        }
      }

      let nx = m.x, ny = m.y;
      if (m.phase === 0) ny = m.y - 1;                                   // climb
      else if (m.phase === 1) {
        if (m.x === m.tx) { m.phase = 2; ny = m.y + 1; }                 // above target: dive
        else nx = m.x + (m.tx > m.x ? 1 : -1);                           // cruise sideways
      } else {                                                           // dive, still homing on the target
        ny = m.y + 1;
        const sel = missileTargets.get(m.mid);
        if (targetAlive(sel) && sel.x !== m.x && Math.abs(sel.x - m.x) <= 25) {
          nx = m.x + (sel.x > m.x ? 1 : -1);                             // steer sideways while falling
        }
      }

      if (outOfBounds(nx, ny)) {
        const x = m.x, y = m.y;
        missileTargets.delete(m.mid);
        deletePixel(x, y);
        if (m.phase === 2) explosion(x, y, MISSILE_RADIUS);
        return;
      }
      const t = pixelMap[nx][ny];
      if (t && t !== m) {
        if (t.element === "warrior_missile_flame") {
          deletePixel(nx, ny);                                           // brush past our own flame
        } else if (t.element === "warrior_launcher" && m.age < 6) {
          continue;                                                      // just left the launcher
        } else {
          missileTargets.delete(m.mid);
          deletePixel(m.x, m.y);                                         // hit something: boom
          explosion(nx, ny, MISSILE_RADIUS);
          return;
        }
      }

      const ox = m.x, oy = m.y;
      if (tryMove(m, nx, ny)) {
        createPixel("warrior_missile_flame", ox, oy);                    // small exhaust flame
        const f = pixelMap[ox][oy];
        if (f) f.color = Math.random() < 0.5 ? "rgb(255,140,30)" : "rgb(255,220,70)";
      }
      if (m.phase === 0 && m.y <= m.cy) m.phase = 1;                     // reached cruise height
    }
  }

  function launcherTick(L) {
    if (!L.ready) { L.ready = true; L.cd = 60; paintLauncher(L); }
    if (tryMove(L, L.x, L.y + 1)) return;                        // settle on the ground
    if (L.cd > 0) L.cd--;
    if (!warMode || !L.team || L.cd > 0) return;
    if ((pixelTicks + L.x) % 5) return;

    const list = rosterPrev[L.team === "red" ? "blue" : "red"] || [];
    let best = null, bd = Infinity;
    for (const w of list) {
      if (!isWarrior(w) || outOfBounds(w.x, w.y) || pixelMap[w.x][w.y] !== w) continue;
      const d = Math.hypot(w.x - L.x, w.y - L.y);
      if (d >= LAUNCH_MIN && d <= LAUNCH_MAX && d < bd) { bd = d; best = w; }
    }
    for (const v of enemyVehicles(L.team)) {                       // enemy tanks / helicopters too
      if (!vehicleAlive(v)) continue;
      const d = Math.hypot(v.x - L.x, v.y - L.y);
      if (d >= LAUNCH_MIN && d <= LAUNCH_MAX && d < bd) { bd = d; best = v; }
    }
    if (!best) return;

    const x = L.x, y = L.y - 1;
    if (!isEmpty(x, y)) return;
    createPixel("warrior_missile", x, y);
    const m = pixelMap[x][y];
    if (!m) return;
    m.team = L.team;
    m.phase = 0;                                                 // 0 climb, 1 cruise, 2 dive
    m.cy = Math.max(3, y - MISSILE_RISE);                        // cruise height
    m.tx = best.x;
    m.life = 600;
    m.mid = Math.random().toString(36).slice(2);
    missileTargets.set(m.mid, best);              // its selected target
    L.cd = LAUNCH_CD;
  }

  function step(pixel, dir) {
    if (pixel.mount) return API.horseStep ? API.horseStep(pixel, dir) : false;   // riders steer the horse
    const nx = pixel.x + dir;
    clearBlood(nx, pixel.y); clearBlood(nx, pixel.y - 1); clearBlood(nx, pixel.y - 2);
    const head = getHead(pixel);
    for (const dy of [0, -1]) {          // walk flat, or hop up one pixel
      const ty = pixel.y + dy;
      if (!isEmpty(nx, ty)) continue;
      if (head && !isEmpty(nx, ty - 1)) continue;   // room for the head too
      // never climb over another warrior
      if (dy === -1 && isWarriorPart(pixelMap[nx][pixel.y])) continue;
      if (tryMove(pixel, nx, ty)) {
        if (head) tryMove(head, nx, ty - 1);
        return true;
      }
    }
    return false;
  }

  function warriorTick(pixel) {
    if (!pixel.ready) { init(pixel); ensureHead(pixel); tint(pixel); }
    if (pixel.mount && !(API.horseOfRider && API.horseOfRider(pixel))) pixel.mount = undefined;   // horse is gone
    if (pixel.cd > 0) pixel.cd--;
    register(pixel);
    if (pixel.throwCd > 0) pixel.throwCd--;
    if (pixel.gunCd > 0) pixel.gunCd--;
    if (pixel.flash > 0) { pixel.flash--; if (!pixel.flash) tint(pixel); }
    pickup(pixel);                          // grab gear/potions nearby, including right under us

    // gravity (head follows)
    if (isEmpty(pixel.x, pixel.y + 1)) {
      fall(pixel);
      return;
    }

    // no stacking: if standing on another warrior, slide off to the side
    if (!outOfBounds(pixel.x, pixel.y + 1) && isWarriorPart(pixelMap[pixel.x][pixel.y + 1])) {
      const d = Math.random() < 0.5 ? -1 : 1;
      if (!step(pixel, d)) step(pixel, -d);
      return;
    }

    // heat damage
    if (pixel.temp > 200) {
      pixel.hp -= 1;
      if (pixel.hp <= 0) { changePixel(pixel, "ash"); return; }
    }

    // act every other tick
    if ((pixelTicks + pixel.x) % 2) return;
    ensureHead(pixel);

    // WAR: only teamed warriors fight, and only when war is on.
    // Target is remembered as coordinates only (no pixel references).
    if (!warMode) pixel.fighter = false;
    if (warMode && pixel.team && (pixel.fighter || pixel.spear > 0 || pixel.gun || tryPromote(pixel))) {
      let tgt = null;
      if (pixel.tx !== undefined) {
        for (let dx = -1; dx <= 1 && !tgt; dx++) {
          for (let dy = -1; dy <= 1 && !tgt; dy++) {
            const x = pixel.tx + dx, y = pixel.ty + dy;
            if (outOfBounds(x, y)) continue;
            const c = pixelMap[x][y];
            const cv = API.vehicleOf ? API.vehicleOf(c) : null;
            if (isEnemy(pixel, c)) tgt = c;
            else if (vehicleAlive(cv) && cv.team && cv.team !== pixel.team) tgt = cv;
          }
        }
      }
      pixel.scan = (pixel.scan || 0) + 1;
      if (!tgt && pixel.scan >= SCAN_EVERY) {
        pixel.scan = Math.floor(Math.random() * 3);
        tgt = findTarget(pixel);
        if (!tgt) { pixel.tx = undefined; pixel.ty = undefined; }
      }
      if (tgt) {
        const aim = aimPoint(pixel, tgt);
        pixel.tx = aim.x; pixel.ty = aim.y;
        const ddx = Math.abs(aim.x - pixel.x), ddy = Math.abs(aim.y - pixel.y);
        const rise = (pixel.mount ? 1 : 0) + (tgt.mount ? 1 : 0);     // mounted fighters stand higher up
        // a horse is wide: a mounted fighter (or a mounted target) can't get close, so reach grows by 3
        const extra = (pixel.mount ? 3 : 0) + (tgt.mount ? 3 : 0);
        const reach = (pixel.spear > 0 ? 2 : 1) + extra;
        if (ddx <= reach && ddy <= 1 + rise) {
          attack(pixel, tgt, Math.max(0, ddx - extra));      // distance as seen by the damage code
        } else {
          const dir = aim.x > pixel.x ? 1 : -1;
          pixel.dir = dir;
          // gunner: shoot from range, back off if the enemy gets too close
          const gun = GUNS[pixel.gun];
          if (gun && Math.hypot(ddx, ddy) <= gun.range) {
            shoot(pixel, aim);
            if (Math.hypot(ddx, ddy) <= gun.keep) { if (!step(pixel, -dir)) step(pixel, dir); }
            return;
          }
          // spearman: keep distance and throw; melee only when they get close
          if (pixel.spear > 0 && ddy <= 1 + rise && ddx <= 12) {
            if (ddx >= 3) throwSpear(pixel, aim);
            if (ddx <= 4) { if (!step(pixel, -dir)) step(pixel, dir); }  // back off, or close in if cornered
            return;
          }
          if (!step(pixel, dir)) pixel.dir = -pixel.dir;
        }
        return;
      }
      // target moved out of reach: keep marching to where it was last seen
      if (pixel.tx !== undefined) {
        if (Math.abs(pixel.tx - pixel.x) > 1) {
          const dir = pixel.tx > pixel.x ? 1 : -1;
          pixel.dir = dir;
          step(pixel, dir);
          return;
        }
        pixel.tx = undefined; pixel.ty = undefined;
      }
    } else if (pixel.tx !== undefined) {
      pixel.tx = undefined; pixel.ty = undefined;
    }

    // PEACE (or nobody nearby): mine and wander
    const mineRate = (warMode && pixel.team && !pixel.fighter) ? 0.5 : 0.1;
    if (Math.random() < mineRate && mine(pixel)) return;
    if (Math.random() < 0.05) pixel.dir = -pixel.dir;
    if (Math.random() < 0.6) {
      const ax = pixel.x + pixel.dir;
      const ahead = outOfBounds(ax, pixel.y) ? null : pixelMap[ax][pixel.y];
      if (isWarriorPart(ahead)) pixel.dir = -pixel.dir;      // give friends space
      else if (!step(pixel, pixel.dir)) pixel.dir = -pixel.dir;
    }
  }

  function setTeam(pixel, team) {
    if (API.setVehicleTeam && API.setVehicleTeam(pixel, team)) return;   // vehicles use the same team tools
    if (pixel.element === "warrior_launcher") {
      pixel.team = team;
      paintLauncher(pixel);
      return;
    }
    if (!isWarrior(pixel)) return;
    if (!pixel.ready) init(pixel);
    pixel.team = team;
    pixel.fighter = false;
    pixel.tx = undefined; pixel.ty = undefined;
    tint(pixel);
  }

  function colorProps(rgb) {
    return {
      color: "#" + rgb.map(v => v.toString(16).padStart(2, "0")).join(""),
      // Sandboxels builds this at startup; console-loaded mods must set it
      colorObject: { r: rgb[0], g: rgb[1], b: rgb[2] },
    };
  }

  // The warrior itself (neutral)
  elements.warrior = Object.assign({
    name: "Warrior",
    category: "life",
    state: "solid",
    density: 1000,
    temp: 36,
    tempHigh: 400,
    stateHigh: "ash",
    burn: 10,
    burnTime: 100,
    burnInto: "ash",
    tick: warriorTick,
    hardness: 0.3,
    breakInto: "blood",
    desc: "Mines iron and wood and crafts swords + armor. Use the team tools to give it a team.",
  }, colorProps(BASE.none));

  // The head (spawned automatically on top of each warrior, hidden from menu)
  elements.warrior_head = Object.assign({
    name: "Warrior Head",
    category: "life",
    hidden: true,
    excludeRandom: true,
    state: "solid",
    density: 1000,
    temp: 36,
    tempHigh: 400,
    stateHigh: "ash",
    burn: 10,
    burnTime: 100,
    burnInto: "ash",
    tick: headTick,
    hardness: 0.3,
    breakInto: "blood",
    desc: "The head of a Warrior.",
  }, colorProps([232, 185, 138]));

  // Team tools
  const TOOLS = [
    ["warrior_team_red",  "Red Team",  "red",  BASE.red],
    ["warrior_team_blue", "Blue Team", "blue", BASE.blue],
    ["warrior_team_none", "No Team",   null,   BASE.none],
  ];
  for (const [id, label, team, rgb] of TOOLS) {
    elements[id] = Object.assign({
      name: label,
      category: "tools",
      excludeRandom: true,
      tool: function (pixel) { setTeam(pixel, team); },
      desc: "Paint over Warriors to set their team.",
    }, colorProps(rgb));
  }

  // Peace / War buttons in the tools bar: just clicking them sets the mode
  const MODES = [
    ["warrior_peace", "Peace", false, [46, 139, 87]],
    ["warrior_war",   "War",   true,  [192, 57, 43]],
  ];
  for (const [id, label, mode, rgb] of MODES) {
    elements[id] = Object.assign({
      name: label,
      category: "tools",
      excludeRandom: true,
      onSelect: function () { warMode = mode; refreshBtn(); },
      tool: function () {},
      desc: mode ? "Red and Blue teams fight each other." : "Everyone gets along.",
    }, colorProps(rgb));
  }

  // Gear items: place them in the world, warriors pick them up.
  // They fall using their own tick (no built-in behavior, which console-loaded
  // mods don't get set up for).
  function itemTick(p) {
    if (tryMove(p, p.x, p.y + 1)) return;
    const d = Math.random() < 0.5 ? -1 : 1;
    if (isEmpty(p.x + d, p.y) && isEmpty(p.x + d, p.y + 1)) tryMove(p, p.x + d, p.y + 1);
  }
  // Spear item: a "--" made of two brown pixels that fall together
  function spearItemTick(p) {
    let t = outOfBounds(p.x + 1, p.y) ? null : pixelMap[p.x + 1][p.y];
    if (!(t && t.element === "warrior_spear_tail")) {
      t = null;
      if (!p.tailed && isEmpty(p.x + 1, p.y)) {
        createPixel("warrior_spear_tail", p.x + 1, p.y);
        t = pixelMap[p.x + 1][p.y];
      }
      p.tailed = true;
    }
    if (isEmpty(p.x, p.y + 1) && (!t || isEmpty(t.x, t.y + 1))) {
      const ty = t ? t.y : 0;
      tryMove(p, p.x, p.y + 1);
      if (t) tryMove(t, t.x, ty + 1);
    }
  }
  function spearTailTick(p) {
    const left = outOfBounds(p.x - 1, p.y) ? null : pixelMap[p.x - 1][p.y];
    if (left && left.element === "warrior_spear") { p.orph = 0; return; }
    p.orph = (p.orph || 0) + 1;
    if (p.orph > 2) deletePixel(p.x, p.y);     // spear was picked up
  }

  const GEAR = [
    ["warrior_sword", "Sword", [200, 210, 225], "Warriors pick this up to hit harder (up to 3 swords)."],
    ["warrior_armor", "Armor", [130, 140, 155], "Warriors pick this up to take less damage (up to 3 pieces)."],
    ["warrior_spear", "Spear", [139, 90, 43], "Reaches 2 pixels, weaker than a sword. Holders keep their distance and throw it endlessly."],
    ["warrior_shield", "Shield", [90, 110, 170], "Gives a chance to block a hit completely (up to 2 shields)."],
  ];
  for (const [id, label, rgb, desc] of GEAR) {
    elements[id] = Object.assign({
      name: label,
      category: "weapons",
      state: "solid",
      density: 3000,
      temp: 20,
      tick: id === "warrior_spear" ? spearItemTick : itemTick,
      hardness: 0.6,
      desc: desc,
    }, colorProps(rgb));
  }

  // Healing Potion: restores 20% of max health, can't go over the max.
  // Hurt warriors drink it when they touch it; healthy ones leave it alone.
  elements.warrior_potion = Object.assign({
    name: "Healing Potion", category: "life", state: "solid", density: 2000, temp: 20,
    tick: itemTick, hardness: 0.2, breakInto: "water",
    desc: "Heals a Warrior for 20% of max health (4 HP). Never goes over the max.",
  }, colorProps([235, 60, 130]));

  // Gun items (place them; warriors pick up the best one) + "Give" tools
  for (const k in GUNS) {
    const g = GUNS[k];
    elements["warrior_" + k] = Object.assign({
      name: g.label, category: "weapons", state: "solid", density: 4000, temp: 20,
      tick: itemTick, hardness: 0.6,
      desc: g.label + ": infinite ammo, range " + g.range + ", damage " + g.dmg +
            (g.pellets.length > 1 ? " per pellet" : "") + ". Holders keep their distance.",
    }, colorProps(g.rgb));
    elements["warrior_give_" + k] = Object.assign({
      name: "Give " + g.label, category: "tools", excludeRandom: true,
      tool: function (pixel) {
        if (!isWarrior(pixel)) return;
        if (!pixel.ready) init(pixel);
        pixel.gun = k;
      },
      desc: "Paint over Warriors to hand them a " + g.label + ".",
    }, colorProps(g.rgb));
  }
  elements.warrior_bullet = Object.assign({
    name: "Bullet", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 4000, temp: 20, tick: bulletTick, desc: "A bullet in flight.",
  }, colorProps([255, 252, 215]));
  elements.warrior_bullet_trail = Object.assign({
    name: "Bullet Trail", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 4000, temp: 20,
    tick: function (p) { p.life = (p.life || 0) + 1; if (p.life >= 3) deletePixel(p.x, p.y); },
    desc: "Tracer behind a bullet.",
  }, colorProps([200, 140, 40]));

  elements.warrior_launcher = Object.assign({
    name: "Missile Launcher", category: "weapons", state: "solid", density: 5000, temp: 20,
    tick: launcherTick, hardness: 0.7,
    desc: "Paint it with Red/Blue Team. In war it fires missiles at the nearest enemy warrior.",
  }, colorProps(LAUNCHER_RGB.none));
  elements.warrior_missile = Object.assign({
    name: "Missile", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 5000, temp: 20, tick: missileTick, desc: "A missile in flight.",
  }, colorProps([210, 210, 220]));
  elements.warrior_missile_flame = Object.assign({
    name: "Missile Flame", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 5000, temp: 20,
    tick: function (p) { p.life = (p.life || 0) + 1; if (p.life >= 3) deletePixel(p.x, p.y); },
    desc: "Exhaust flame behind a missile.",
  }, colorProps([255, 140, 30]));

  // Tool: paint over warriors to hand them spears right away
  elements.warrior_give_spear = Object.assign({
    name: "Give Spears",
    category: "tools",
    excludeRandom: true,
    tool: function (pixel) {
      if (!isWarrior(pixel)) return;
      if (!pixel.ready) init(pixel);
      pixel.spear = MAX_SPEARS;
    },
    desc: "Paint over Warriors to give them 3 spears.",
  }, colorProps([160, 120, 70]));

  // Flying spear (spawned when a warrior throws; hidden from the menu)
  elements.warrior_spear_proj = Object.assign({
    name: "Thrown Spear",
    category: "weapons",
    hidden: true,
    excludeRandom: true,
    state: "solid",
    density: 3000,
    temp: 20,
    tick: projTick,
    desc: "A spear in flight.",
  }, colorProps([139, 90, 43]));

  elements.warrior_spear_trail = Object.assign({
    name: "Spear Trail", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 3000, temp: 20,
    tick: function (p) { p.life = (p.life || 0) + 1; if (p.life >= 12) deletePixel(p.x, p.y); },
    desc: "The back half of a flying spear.",
  }, colorProps([139, 90, 43]));
  elements.warrior_spear_tail = Object.assign({
    name: "Spear Shaft", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 3000, temp: 20, tick: spearTailTick,
    desc: "The back half of a Spear item.",
  }, colorProps([139, 90, 43]));

  // Make sure the buttons exist (harmless if the game already made them)
  try {
    ["warrior", "warrior_team_red", "warrior_team_blue", "warrior_team_none",
     "warrior_peace", "warrior_war", "warrior_sword", "warrior_armor",
     "warrior_spear", "warrior_shield", "warrior_give_spear", "warrior_potion", "warrior_launcher"]
      .concat(Object.keys(GUNS).map(k => "warrior_" + k), Object.keys(GUNS).map(k => "warrior_give_" + k)).forEach(n => {
      if (typeof createElementButton === "function" &&
          !document.getElementById("elementButton-" + n)) {
        createElementButton(n);
      }
    });
  } catch (e) { console.warn("Warriors: button setup", e); }

  // Peace / War toggle button
  const old = document.getElementById("warriorWarBtn");
  if (old) old.remove();
  const btn = document.createElement("button");
  btn.id = "warriorWarBtn";
  btn.style.cssText =
    "position:fixed;top:8px;right:8px;z-index:99999;padding:10px 16px;" +
    "font:bold 16px monospace;border:3px solid #000;border-radius:6px;cursor:pointer;color:#fff;";
  function refreshBtn() {
    btn.textContent = warMode ? "\u2694 WAR" : "\u262E PEACE";
    btn.style.background = warMode ? "#c0392b" : "#2e8b57";
  }
  btn.onclick = function () { warMode = !warMode; refreshBtn(); };
  refreshBtn();
  document.body.appendChild(btn);
  // keep it attached and on top even if the page rebuilds
  clearInterval(window.__warriorBtnTimer);
  window.__warriorBtnTimer = setInterval(function () {
    if (!document.body.contains(btn)) document.body.appendChild(btn);
  }, 2000);
})();

(function () {
  const TEAM_RGB = { none: [135, 135, 140], red: [190, 70, 60], blue: [60, 110, 190] };
  const KIND_RGB = { dark: [55, 55, 60], glass: [150, 200, 230], rotor: [200, 200, 205], barrel: [40, 40, 45], horse: [140, 90, 50], hoof: [70, 45, 25] };
  const CORES = ["veh_tank", "veh_heli", "veh_plane", "veh_horse"];
  const GALLOP = 4;            // cells per rider step = 2x walking speed (an even count keeps the rider acting every other tick)

  function teamRGB(t) { return TEAM_RGB[t || "none"]; }
  function rgbStr(a) { return "rgb(" + a.join(",") + ")"; }
  function vehicleOf(p) {
    if (!p) return null;
    if (p.element === "veh_part") return coreOfPart(p);
    return isCore(p) ? p : null;
  }
  function isCore(p) { return !!p && CORES.indexOf(p.element) >= 0; }
  function W() { return typeof width === "number" ? width : 150; }
  function H() { return typeof height === "number" ? height : 100; }
  function passable(p) { return p.element === "blood" || p.element === "fire" || p.element === "smoke" || p.element === "veh_flame"; }

  // ---------- war / peace: shared with the warriors' PEACE/WAR button ----------
  const API = window.__warVehicleAPI;
  function warActive() { return API.isWar(); }

  // ---------- vehicle shapes (dx is "forward", flipped when facing left) ----------
  function row(dy, from, to, k) {
    const a = [];
    for (let dx = from; dx <= to; dx++) a.push({ dx: dx, dy: dy, k: k });
    return a;
  }
  const TPL = {
    veh_tank: [].concat(
      row(0, -3, -1, "dark"), row(0, 1, 3, "dark"),      // tracks (core is dx 0)
      row(-1, -3, 3, "team"),                              // hull
      row(-2, -1, 1, "team"),                              // turret
      row(-2, 2, 4, "barrel")                              // barrel
    ),
    veh_heli: [].concat(
      row(0, -5, -1, "team"), [{ dx: 1, dy: 0, k: "team" }, { dx: 2, dy: 0, k: "glass" }],
      [{ dx: 0, dy: -1, k: "dark" }, { dx: -5, dy: -1, k: "dark" }],
      row(-2, -3, 3, "rotor"),                             // spinning rotor
      row(1, -1, 1, "dark")                                // skids
    ),
    veh_plane: [].concat(
      row(0, -4, -1, "team"), row(0, 1, 3, "team"), [{ dx: 4, dy: 0, k: "dark" }],
      [{ dx: 1, dy: -1, k: "glass" }, { dx: 2, dy: -1, k: "glass" }, { dx: -1, dy: -1, k: "barrel" }],   // glass cockpit + gun turret
      [{ dx: -4, dy: -1, k: "team" }, { dx: -3, dy: -1, k: "team" }, { dx: -4, dy: -2, k: "team" }],
      row(1, -1, 1, "dark")                                // wings
    ),
    // Horse: 3 legs-row (core is the middle one), body with a team-colored saddle, tail, neck + head.
    // A rider sits at (0,-2) with its head at (0,-3).
    veh_horse: [].concat(
      [{ dx: -2, dy: 0, k: "hoof" }, { dx: 2, dy: 0, k: "hoof" }],
      row(-1, -2, -1, "horse"), [{ dx: 0, dy: -1, k: "team" }], row(-1, 1, 2, "horse"),
      [{ dx: -3, dy: -1, k: "hoof" }],
      [{ dx: 2, dy: -2, k: "horse" }, { dx: 3, dy: -2, k: "horse" }, { dx: 3, dy: -3, k: "hoof" }]
    ),
  };
  const SPEC = {
    veh_tank:  { hp: 200, muzzle: { dx: 5, dy: -2 } },
    veh_heli:  { hp: 120, muzzle: { dx: 3, dy: 0 } },
    veh_plane: { hp: 70,  muzzle: { dx: 5, dy: 0 }, bomb: { dx: 0, dy: 2 }, turret: { dx: -1, dy: -2 } },
    veh_horse: { hp: 40 },
  };
  const SHOT = {
    bullet: { spd: 3, life: 24, r: 0, dW: 6,   dV: 4,  rgb: [255, 230, 90] },
    rocket: { spd: 2, life: 45, r: 3, dW: 40,  dV: 40, rgb: [255, 150, 50] },   // (no longer used by the helicopter)
    minigun:{ spd: 3, life: 30, r: 0, dW: 4,   dV: 3,  rgb: [255, 240, 120] },   // rapid-fire
    turret: { spd: 3, life: 30, r: 0, dW: 8,   dV: 5,  rgb: [255, 150, 60]  },   // plane turret rounds
    shell:  { spd: 3, life: 30, r: 5, dW: 100, dV: 80, rgb: [255, 200, 120] },
    missile:{ spd: 2, life: 70, r: 7, dW: 100, dV: 90, rgb: [215, 215, 225] },   // plane missile: bomb-sized blast
  };

  const targets = new Map();   // vehicle id -> current target pixel (kept out of pixels)
  const missileTargets = new Map();   // guided shot id -> its selected target

  // ---------- parts ----------
  function coreOfPart(p) {
    const cx = p.x - p.ox, cy = p.y - p.oy;
    if (outOfBounds(cx, cy)) return null;
    const c = pixelMap[cx][cy];
    return (c && isCore(c) && c.cid === p.cid) ? c : null;
  }

  function forParts(core, fn) {
    const tpl = TPL[core.type];
    for (const e of tpl) {
      const x = core.x + core.face * e.dx, y = core.y + e.dy;
      if (outOfBounds(x, y)) continue;
      const p = pixelMap[x][y];
      if (p && p.element === "veh_part" && p.cid === core.cid) fn(p, e);
    }
  }

  function colorFor(core, k) {
    return rgbStr(k === "team" ? teamRGB(core.team) : KIND_RGB[k]);
  }

  function build(core) {
    for (const e of TPL[core.type]) {
      const x = core.x + core.face * e.dx, y = core.y + e.dy;
      if (!isEmpty(x, y)) continue;
      createPixel("veh_part", x, y);
      const p = pixelMap[x][y];
      if (!p) continue;
      p.cid = core.cid; p.ox = core.face * e.dx; p.oy = e.dy;
      p.k = e.k; p.team = core.team;
      p.color = colorFor(core, e.k);
    }
  }

  function recolor(core) {
    forParts(core, function (p, e) { p.team = core.team; p.color = colorFor(core, e.k); });
    core.color = core.type === "veh_tank" ? rgbStr(KIND_RGB.dark)
               : core.type === "veh_horse" ? rgbStr(KIND_RGB.hoof)
               : rgbStr(teamRGB(core.team));
  }

  function turn(core, face) {
    forParts(core, function (p) { deletePixel(p.x, p.y); });
    core.face = face;
    build(core);
  }

  // Move the whole vehicle by (mx,my) if every cell it needs is free.
  function moveBy(core, mx, my) {
    const parts = [];
    forParts(core, function (p) { parts.push(p); });
    if (core.type === "veh_horse") {                      // the rider (body + head) moves with the horse
      const r = riderOf(core);
      if (r) { parts.push(r.body); if (r.head) parts.push(r.head); }
    }
    const all = [core].concat(parts);
    const own = new Set(all);
    const clear = [];
    for (const p of all) {
      const nx = p.x + mx, ny = p.y + my;
      if (outOfBounds(nx, ny)) return false;
      const o = pixelMap[nx][ny];
      if (o && !own.has(o)) {
        // our own freshly fired bullets are not obstacles (otherwise a plane "crashes" into its own gunfire)
        if (passable(o) || (o.element === "veh_shot" && o.team === core.team)) clear.push(o); else return false;
      }
    }
    for (const o of clear) deletePixel(o.x, o.y);
    all.sort(function (a, b) { return (b.x * mx + b.y * my) - (a.x * mx + a.y * my); });
    for (const p of all) tryMove(p, p.x + mx, p.y + my);
    return true;
  }

  function supported(core) {                       // is something under the bottom row?
    const tpl = TPL[core.type];
    let maxDy = 0;
    for (const e of tpl) if (e.dy > maxDy) maxDy = e.dy;
    for (const e of tpl) {
      if (e.dy !== maxDy) continue;
      const x = core.x + core.face * e.dx, y = core.y + maxDy + 1;
      if (outOfBounds(x, y) || !isEmpty(x, y)) return true;
    }
    return false;
  }

  // ---------- damage ----------
  function damageWarrior(w, dmg) {
    w.hp = (w.hp === undefined ? 20 : w.hp) - dmg;
    if (w.hp <= 0) {
      if (!outOfBounds(w.x, w.y - 1)) {
        const h = pixelMap[w.x][w.y - 1];
        if (h && h.element === "warrior_head") deletePixel(h.x, h.y);
      }
      changePixel(w, "blood");
    } else {
      w.color = "rgb(255,255,255)"; w.flash = 4;
    }
  }

  function destroy(c) {
    if (c.dead) return;
    c.dead = true;
    forParts(c, function (p) { deletePixel(p.x, p.y); });
    targets.delete(c.cid);
    const x = c.x, y = c.y, team = c.team;
    deletePixel(x, y);
    if (c.type === "veh_horse") createPixel("blood", x, y);   // a horse just collapses
    else blast(x, y, 5, 60, 0, team);
  }

  function damageVeh(c, dmg) {
    if (c.dead) return;
    c.hp -= dmg;
    if (c.hp <= 0) destroy(c);
  }

  function blast(x, y, r, dW, dV, team) {
    const hitW = new Set(), hitV = new Set();
    for (let dx = -r; dx <= r; dx++) {
      for (let dy = -r; dy <= r; dy++) {
        const d = Math.hypot(dx, dy);
        if (d > r) continue;
        const cx = x + dx, cy = y + dy;
        if (outOfBounds(cx, cy)) continue;
        const p = pixelMap[cx][cy];
        if (p) {
          if (p.element === "warrior") { if (!(team && p.team === team)) hitW.add(p); continue; }
          if (p.element === "warrior_head") {
            const b = outOfBounds(p.x, p.y + 1) ? null : pixelMap[p.x][p.y + 1];
            if (b && b.element === "warrior" && !(team && b.team === team)) hitW.add(b);
            continue;
          }
          if (p.element === "veh_part") { const c = coreOfPart(p); if (c && !(team && c.team === team)) hitV.add(c); continue; }
          if (isCore(p)) { if (!(team && p.team === team)) hitV.add(p); continue; }
          const info = elements[p.element];
          if (info && (info.hardness || 0) >= 0.9) continue;
          if (Math.random() < 1 - 0.6 * (d / r)) deletePixel(cx, cy);
        } else if (d > r * 0.5 && Math.random() < 0.1) {
          createPixel("fire", cx, cy);
        }
      }
    }
    for (const w of hitW) {
      if (!outOfBounds(w.x, w.y) && pixelMap[w.x][w.y] === w) damageWarrior(w, dW);
    }
    for (const c of hitV) if (dV > 0) damageVeh(c, dV);
  }

  // ---------- shooting ----------
  function fire(c, kind, off, tx, ty, selected) {
    const x = c.x + c.face * off.dx, y = c.y + off.dy;
    if (!isEmpty(x, y)) return false;
    createPixel("veh_shot", x, y);
    const s = pixelMap[x][y];
    if (!s) return false;
    const P = SHOT[kind];
    let dx = tx - x, dy = ty - y;
    if (kind === "missile") { dx = 0; dy = 1; }   // dropped straight down
    const m = Math.hypot(dx, dy) || 1;
    s.kind = kind; s.team = c.team;
    s.fx = x; s.fy = y; s.ux = dx / m; s.uy = dy / m;
    s.spd = P.spd; s.life = P.life; s.r = P.r; s.dW = P.dW; s.dV = P.dV;
    s.color = rgbStr(P.rgb);
    if (kind === "rocket" || kind === "missile") {            // guided: lock onto the selected target
      s.mid = Math.random().toString(36).slice(2);
      if (selected) missileTargets.set(s.mid, selected);
    }
    return true;
  }

  function endShot(s) { if (s.mid !== undefined) missileTargets.delete(s.mid); }

  function shotTick(s) {
    if (s.life === undefined) return;
    if (--s.life < 0) {
      endShot(s);
      const x = s.x, y = s.y;
      deletePixel(x, y);
      if (s.r > 0) blast(x, y, s.r, s.dW, s.dV, s.team);
      return;
    }
    if (s.mid !== undefined) {                 // AIMBOT: steer toward the selected target every tick
      let tg = missileTargets.get(s.mid);
      if (!isLive(tg) && (pixelTicks + s.x) % 6 === 0) {     // lost it: lock onto the nearest enemy
        tg = findTarget(s);
        if (tg) missileTargets.set(s.mid, tg);
      }
      if (isLive(tg)) {
        const dx = tg.x - s.fx, dy = tg.y - s.fy, m = Math.hypot(dx, dy) || 1;
        s.ux = dx / m; s.uy = dy / m;
      }
    }
    let lx = s.x, ly = s.y;
    for (let i = 0; i < s.spd; i++) {
      s.fx += s.ux; s.fy += s.uy;
      const cx = Math.round(s.fx), cy = Math.round(s.fy);
      if (cx === lx && cy === ly) continue;
      if (outOfBounds(cx, cy)) { endShot(s); deletePixel(s.x, s.y); return; }
      const t = pixelMap[cx][cy];
      if (t && t !== s) {
        if (passable(t) || t.element === "veh_shot") continue;
        let wbody = null, owner = null;
        if (t.element === "warrior") wbody = t;
        else if (t.element === "warrior_head") {
          const b = outOfBounds(t.x, t.y + 1) ? null : pixelMap[t.x][t.y + 1];
          if (b && b.element === "warrior") wbody = b;
        } else if (t.element === "veh_part") owner = coreOfPart(t);
        else if (isCore(t)) owner = t;
        if (s.team && ((wbody && wbody.team === s.team) || (owner && owner.team === s.team))) continue;   // friends: pass
        endShot(s);
        deletePixel(s.x, s.y);
        if (s.r > 0) blast(lx, ly, s.r, s.dW, s.dV, s.team);     // explode right at the point of impact
        else { if (wbody) damageWarrior(wbody, s.dW); if (owner) damageVeh(owner, s.dV); }
        return;
      }
      lx = cx; ly = cy;
    }
    if (lx !== s.x || ly !== s.y) {
      const ox = s.x, oy = s.y;
      if (tryMove(s, lx, ly) && s.kind === "missile" && isEmpty(ox, oy)) {
        createPixel("veh_flame", ox, oy);
        const f = pixelMap[ox][oy];
        if (f) f.color = Math.random() < 0.5 ? "rgb(255,140,30)" : "rgb(255,220,70)";
      }
    }
  }

  // ---------- targeting ----------
  function isLive(t) {
    return t && !outOfBounds(t.x, t.y) && pixelMap[t.x][t.y] === t && (t.element === "warrior" || isCore(t));
  }

  function findTarget(c) {
    if (typeof currentPixels === "undefined") return null;
    let best = null, bd = Infinity;
    for (const p of currentPixels) {
      if (!p || !p.team || p.team === c.team) continue;
      if (p.element !== "warrior" && !isCore(p)) continue;
      const d = Math.hypot(p.x - c.x, p.y - c.y);
      if (d < bd) { bd = d; best = p; }
    }
    return best;
  }

  function getTarget(c) {
    let t = targets.get(c.cid);
    if (!isLive(t)) t = null;
    if (!t || (pixelTicks + c.x) % 15 === 0) {
      const f = findTarget(c);
      if (f) t = f;
      targets.set(c.cid, t);
    }
    return t;
  }

  // ---------- roster of tanks/helis for the warriors to target (planes are left out on purpose) ----------
  let vrTick = -1, vrCur = { red: [], blue: [] }, vrPrev = { red: [], blue: [] };
  function registerVeh(c) {
    if (!c.team || !vrCur[c.team] || c.element === "veh_plane") return;
    if (vrTick !== pixelTicks) { vrPrev = vrCur; vrCur = { red: [], blue: [] }; vrTick = pixelTicks; }
    vrCur[c.team].push(c);
  }

  // ---------- vehicle setup ----------
  function groundBelow(c) {
    for (let y = c.y + 2; y < H(); y++) if (!isEmpty(c.x, y)) return y;
    return H();
  }

  function init(c) {
    c.ready = true;
    c.type = c.element;
    c.cid = Math.random().toString(36).slice(2);
    c.hp = SPEC[c.type].hp;
    c.team = c.team || null;
    c.face = c.x < W() / 2 ? 1 : -1;
    c.cd = 0; c.gunCd = 0; c.turretCd = 0; c.turnCd = 0;
    if (c.type === "veh_heli") c.altY = Math.max(6, Math.min(c.y, groundBelow(c) - 18));
    if (c.type === "veh_plane") c.altY = Math.max(6, Math.min(c.y, groundBelow(c) - 28));
    build(c);
    recolor(c);
  }

  function prep(c) {
    if (!c.ready) init(c);
    registerVeh(c);
    if (c.cd > 0) c.cd--;
    if (c.gunCd > 0) c.gunCd--;
    if (c.turretCd > 0) c.turretCd--;
    if (c.turnCd > 0) c.turnCd--;
  }

  // ---------- TANK ----------
  function tankTick(c) {
    prep(c);
    if (!supported(c)) { moveBy(c, 0, 1); return; }       // gravity
    if ((pixelTicks + c.x) % 3) return;                    // slow and heavy
    if (!(warActive() && c.team)) return;
    const t = getTarget(c);
    if (!t) return;
    const dx = t.x - c.x, dy = t.y - c.y, dist = Math.hypot(dx, dy);
    const want = dx >= 0 ? 1 : -1;
    if (want !== c.face) {
      if (c.turnCd <= 0) { turn(c, want); c.turnCd = 30; }
      return;
    }
    if (dist <= 60 && c.cd <= 0) {
      if (fire(c, "shell", SPEC.veh_tank.muzzle, t.x, t.y)) c.cd = 100;
    }
    if (dist > 35) {                                       // drive closer, climb 1-pixel steps
      if (!moveBy(c, c.face, 0)) moveBy(c, c.face, -1);
    }
  }

  // ---------- HELICOPTER ----------
  function heliTick(c) {
    prep(c);
    if ((pixelTicks + c.x) % 2) return;
    const t = (warActive() && c.team) ? getTarget(c) : null;
    let mx = 0, my = 0;
    if (t) {
      const dx = t.x - c.x, want = dx >= 0 ? 1 : -1;
      if (want !== c.face) {
        if (c.turnCd <= 0) { turn(c, want); c.turnCd = 25; }
        return;
      }
      const adx = Math.abs(dx);
      if (adx > 25) mx = want; else if (adx < 12) mx = -want;
    }
    if (c.y > c.altY + 1) my = -1; else if (c.y < c.altY - 1) my = 1;
    if (mx || my) {
      if (!moveBy(c, mx, my)) {
        if (mx && my) { if (!moveBy(c, 0, my)) moveBy(c, mx, 0); }
        else if (mx) moveBy(c, 0, -1);                    // blocked sideways: climb
      }
    }
    if (c.dead || !t) return;
    const dist = Math.hypot(t.x - c.x, t.y - c.y);
    if (dist <= 55 && c.gunCd <= 0) {                       // minigun only: rapid fire, no missiles
      if (fire(c, "minigun", SPEC.veh_heli.muzzle, t.x, t.y)) c.gunCd = 3;
    }
  }

  // ---------- PLANE ----------
  // Returns the enemy warrior/vehicle below (within 3 columns), or null. The missile locks onto it.
  function overEnemy(c) {
    for (let dx = -3; dx <= 3; dx++) {
      const x = c.x + dx;
      for (let y = c.y + 2; y < H(); y++) {
        if (outOfBounds(x, y)) break;
        const p = pixelMap[x][y];
        if (!p || passable(p)) continue;
        let wb = null;
        if (p.element === "warrior") wb = p;
        else if (p.element === "warrior_head") {                 // heads have no team: ask the body
          const b2 = outOfBounds(p.x, p.y + 1) ? null : pixelMap[p.x][p.y + 1];
          if (b2 && b2.element === "warrior") wb = b2;
        }
        if (wb && wb.team && wb.team !== c.team) return wb;
        const veh = vehicleOf(p);
        if (veh && veh.team && veh.team !== c.team) return veh;
        break;                                                   // something else is in the way
      }
    }
    return null;
  }

  function planeTick(c) {
    prep(c);
    const fighting = warActive() && c.team;
    const t = fighting ? getTarget(c) : null;

    if (c.turnCd <= 0) {
      if (c.face > 0 && c.x > W() - 14) { turn(c, -1); c.turnCd = 30; return; }
      if (c.face < 0 && c.x < 14)       { turn(c, 1);  c.turnCd = 30; return; }
      if (t) {
        const dx = t.x - c.x;
        if ((c.face > 0 && dx < -40) || (c.face < 0 && dx > 40)) { turn(c, -c.face); c.turnCd = 30; return; }
      }
    }

    let my = 0;
    if (pixelTicks % 4 === 0) { if (c.y > c.altY + 1) my = -1; else if (c.y < c.altY - 1) my = 1; }
    if (!moveBy(c, c.face, my)) {
      if (!moveBy(c, c.face, -1)) {                        // blocked: hit something, bounce off
        damageVeh(c, 10);
        if (c.dead) return;
        if (c.turnCd <= 0) { turn(c, -c.face); c.turnCd = 30; }
      }
    }
    if (c.dead || !t) return;

    const dx = t.x - c.x, dist = Math.hypot(dx, t.y - c.y);
    // missile every 5 seconds (150 ticks), only while flying over enemy warriors or vehicles
    if (c.cd <= 0 && pixelTicks % 2 === 0) {
      const sel = overEnemy(c);
      if (sel && fire(c, "missile", SPEC.veh_plane.bomb, sel.x, sel.y, sel)) c.cd = 150;
    }
    if (c.gunCd <= 0 && dx * c.face > 0 && dist <= 35) {               // nose minigun: forward only
      if (fire(c, "minigun", SPEC.veh_plane.muzzle, t.x, t.y)) c.gunCd = 4;
    }
    if (c.turretCd <= 0 && dist <= 40) {                               // turret: swivels to hit anything, any direction
      if (fire(c, "turret", SPEC.veh_plane.turret, t.x, t.y)) c.turretCd = 10;
    }
  }

  // ---------- HORSE ----------
  function riderOf(c) {
    if (outOfBounds(c.x, c.y - 2)) return null;
    const b = pixelMap[c.x][c.y - 2];
    if (!(b && b.element === "warrior" && b.mount === c.cid)) return null;
    const h = outOfBounds(c.x, c.y - 3) ? null : pixelMap[c.x][c.y - 3];
    return { body: b, head: (h && h.element === "warrior_head") ? h : null };
  }

  // put a warrior (body + head) onto the saddle
  function mountOn(c, w) {
    const sx = c.x, sy = c.y - 2;
    if (w.x === sx && w.y === sy) { w.mount = c.cid; return true; }       // already on top
    const hh = outOfBounds(w.x, w.y - 1) ? null : pixelMap[w.x][w.y - 1];
    const head = (hh && hh.element === "warrior_head") ? hh : null;
    if (!isEmpty(sx, sy) || !isEmpty(sx, sy - 1)) return false;
    if (!tryMove(w, sx, sy)) return false;
    if (head) tryMove(head, sx, sy - 1);
    w.mount = c.cid;
    return true;
  }

  function tryMountNearby(c) {
    let best = null, bd = Infinity;
    for (let dx = -4; dx <= 4; dx++) {
      for (let dy = -4; dy <= 1; dy++) {
        const x = c.x + dx, y = c.y + dy;
        if (outOfBounds(x, y)) continue;
        const w = pixelMap[x][y];
        if (w && w.element === "warrior" && !w.mount) {
          const d = Math.abs(dx) + Math.abs(dy);
          if (d < bd) { bd = d; best = w; }
        }
      }
    }
    return best ? mountOn(c, best) : false;
  }

  function horseTick(c) {
    prep(c);
    if (!supported(c)) { moveBy(c, 0, 1); return; }               // gravity (rider comes along)
    const r = riderOf(c);
    if (r) {                                                       // ridden: the rider's AI steers us
      const t = r.body.team || null;
      if (c.team !== t) { c.team = t; recolor(c); }
      return;
    }
    if (c.team) { c.team = null; recolor(c); }                     // no rider, no team
    if (pixelTicks % 5 === 0 && tryMountNearby(c)) return;         // a warrior nearby hops on
    if ((pixelTicks + c.x) % 8 === 0 && Math.random() < 0.5) {     // idle: wander a little
      const dir = Math.random() < 0.5 ? -1 : 1;
      if (c.face !== dir) { turn(c, dir); return; }
      if (!moveBy(c, dir, 0)) moveBy(c, dir, -1);
    }
  }

  // ---------- part upkeep ----------
  function partTick(p) {
    const c = coreOfPart(p);
    if (!c) {
      p.orph = (p.orph || 0) + 1;
      if (p.orph > 3) deletePixel(p.x, p.y);
      return;
    }
    p.orph = 0;
    if (p.k === "rotor") p.color = ((pixelTicks >> 1) % 2) ? "rgb(210,210,215)" : "rgb(85,85,90)";
  }

  // ---------- elements ----------
  function colorProps(rgb) {
    return {
      color: "#" + rgb.map(function (v) { return v.toString(16).padStart(2, "0"); }).join(""),
      colorObject: { r: rgb[0], g: rgb[1], b: rgb[2] },     // needed for console-loaded mods
    };
  }

  const VEHICLES = [
    ["veh_tank",  "Tank",       tankTick,  [80, 100, 70],   "Drives toward the enemy and fires explosive shells. Paint a team on it."],
    ["veh_heli",  "Helicopter", heliTick,  [120, 125, 130], "Hovers, keeps its distance and strafes with a rapid-fire minigun."],
    ["veh_plane", "Plane",      planeTick, [150, 150, 155], "Flies fast passes with a nose minigun and a swiveling turret, and fires a big missile every 5 seconds when over enemy warriors or tanks."],
    ["veh_horse", "Horse",      horseTick, [140, 90, 50],   "A warrior standing nearby climbs on and rides it at double speed. It takes the rider's team."],
  ];
  for (const v of VEHICLES) {
    elements[v[0]] = Object.assign({
      name: v[1], category: "weapons", state: "solid", density: 6000, temp: 20,
      tick: v[2], hardness: 0.95, desc: v[4],
    }, colorProps(v[3]));
  }
  elements.veh_part = Object.assign({
    name: "Vehicle Part", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 6000, temp: 20, tick: partTick, hardness: 0.95,
    desc: "A piece of a vehicle.",
  }, colorProps([100, 100, 105]));
  elements.veh_shot = Object.assign({
    name: "Vehicle Shot", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 6000, temp: 20, tick: shotTick,
    desc: "A bullet, rocket, shell or bomb.",
  }, colorProps([255, 230, 90]));

  elements.veh_flame = Object.assign({
    name: "Missile Flame", category: "weapons", hidden: true, excludeRandom: true,
    state: "solid", density: 6000, temp: 20,
    tick: function (p) { p.life = (p.life || 0) + 1; if (p.life >= 3) deletePixel(p.x, p.y); },
    desc: "Exhaust flame behind a missile.",
  }, colorProps([255, 140, 30]));

  // Hooks the warriors code uses: the Red/Blue/No Team tools also set vehicle teams,
  // and warrior bullets/missiles can damage vehicles.
  elements.veh_horse.category = "life";
  API.vehicleOf = vehicleOf;
  API.enemyVehicles = function (myTeam) { return vrPrev[myTeam === "red" ? "blue" : "red"] || []; };
  API.nearestPart = function (core, x, y) {                      // closest pixel of a vehicle to (x,y)
    if (!core.ready) return core;
    let best = core, bd = Math.hypot(core.x - x, core.y - y);
    forParts(core, function (p) {
      const d = Math.hypot(p.x - x, p.y - y);
      if (d < bd) { bd = d; best = p; }
    });
    return best;
  };
  API.damageVeh = damageVeh;
  API.horseOfRider = function (w) {                              // the horse this warrior is sitting on
    if (!w.mount || outOfBounds(w.x, w.y + 2)) return null;
    const c = pixelMap[w.x][w.y + 2];
    return (c && c.element === "veh_horse" && c.cid === w.mount && !c.dead) ? c : null;
  };
  API.horseStep = function (w, dir) {                           // a rider steers: gallop = 2 cells per step
    const c = API.horseOfRider(w);
    if (!c) return false;
    if (c.face !== dir) { turn(c, dir); return true; }
    let moved = false;
    for (let i = 0; i < GALLOP; i++) { if (moveBy(c, dir, 0) || moveBy(c, dir, -1)) moved = true; else break; }
    return moved;
  };
  API.setVehicleTeam = function (pixel, team) {
    const c = vehicleOf(pixel);
    if (!c) return false;
    if (!c.ready) init(c);
    if (c.type === "veh_horse") {                                // a horse takes its rider's team
      const r = riderOf(c);
      if (r && API.setTeam) API.setTeam(r.body, team);
      return true;
    }
    c.team = team;
    recolor(c);
    return true;
  };

  try {
    ["veh_tank", "veh_heli", "veh_plane", "veh_horse"].forEach(function (n) {
      if (typeof createElementButton === "function" && !document.getElementById("elementButton-" + n)) {
        createElementButton(n);
      }
    });
  } catch (e) { console.warn("Vehicles: button setup", e); }
})();
