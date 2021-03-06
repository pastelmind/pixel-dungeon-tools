# Fly Farming Simulator

<template class="style-placeholder">
  <style>
    #strength {
      width: 3em;
    }

    #output_table table {
      border-collapse: collapse;
    }

    #output_table {
      margin: 1em;
    }

    #output_table td, #output_table th {
      border: 1px solid black;
      padding: .2em .4em;
    }

    #output_table tr:nth-child(2n) {
      background-color: #888;
    }
  </style>
</template>

This tool estimates the number of Potions of Healing you can get by killing a Swarm of Flies.

It makes the following assumptions:

* Initially, the Swarm of Flies has maximum HP (80).
* There is always enough room for the Swarm of Flies to split every time.
* The player uses a single weapon to attack the Swarm until it and all of its clones are dead.
* The player has no weapon or armor enchantments, or any damage modifiers other than your Strength and weapon upgrades.

<fieldset id="swarm_form">
  <legend>Parameters</legend>
  <div>
    <label>Your strength: <input id="strength" type="number" min="0" max="20" value="10"> points</label>
  </div>
  <div>
    <label>Weapon: <select id="weapons"></select></label>
  </div>
  <div>
    <label><input id="is_huntress" type="checkbox"> Is Huntress?</label>
  </div>
</fieldset>

<div id="output_table"></div>

<script>

document.head.appendChild(
  document.querySelector('template.style-placeholder').content
);

/**
 * Generates a melee weapon template using the given parameters.
 */
function generateMeleeWeapon(name, tier, accuracy, attackDelay) {
  return {
    name,
    minDamage: tier,
    maxDamage: Math.floor((tier * tier - tier + 10) / accuracy * attackDelay),
    minDamagePerLevel: 1,
    maxDamagePerLevel: tier,
    strengthRequirement: 8 + tier * 2,
    isRanged: false,
  };
}

const weapons = {
  knuckleduster: generateMeleeWeapon('Knuckleduster', 1, 1, 0.5),
  dagger: generateMeleeWeapon('Dagger', 1, 1.2, 1),
  short_sword: Object.assign(
    generateMeleeWeapon('Short Sword', 1, 1, 1),
    { maxDamage: 12, strengthRequirement: 11, }
  ),
  quarterstaff: generateMeleeWeapon('Quarterstaff', 2, 1, 1),
  spear: generateMeleeWeapon('Spear', 2, 1, 1.5),
  mace: generateMeleeWeapon('Mace', 3, 1, 0.8),
  sword: generateMeleeWeapon('Sword', 3, 1, 1),
  battle_axe: generateMeleeWeapon('Battle Axe', 4, 1.2, 1),
  longsword: generateMeleeWeapon('Longsword', 4, 1, 1),
  war_hammer: generateMeleeWeapon('War Hammer', 5, 1.2, 1),
  glaive: generateMeleeWeapon('Glaive', 5, 1, 1),
  'boomerang': {
    name: 'Boomerang',
    minDamage: 1,
    maxDamage: 4,
    minDamagePerLevel: 1,
    maxDamagePerLevel: 2,
    strengthRequirement: 10,
    isRanged: true,
  },
  'wand': {
    name: 'Wand (melee)',
    minDamage(level) { return 1 + Math.floor(level / 3); },
    maxDamage(level) {
      const tier = this.minDamage(level);
      return Math.floor((tier * tier - tier + 10) / 2) + level;
    },
  },
};


/**
 * Creates a damage distribution that matches com.watabou.utils.Random.IntRange
 */
function uniformIntDistribution(min, max) {
  if (min > max)
    throw new Error(`min={min} must be lesser than max={max}`);

  const results = new Map;
  const chance = 1 / (max - min + 1);

  for (let dmg = min; dmg <= max; ++dmg)
    results.set(dmg, chance);

  return results;
}


/**
 * Creates a damage distribution that matches com.watabou.utils.Random.NormalIntRange
 */
function triangularIntDistribution(min, max) {
  if (min > max)
    throw new Error(`min={min} must be lesser than max={max}`);

  const half_width = (max - min + 1) / 2;
  const height = 1 / half_width;
  const slope = 1 / (half_width * half_width);

  const results = new Map;
  for (let dmg = min; dmg <= max; ++dmg) {
    const x = dmg - min;
    if (x < half_width) {
      if (x + 1 > half_width)
        results.set(dmg, height - slope / 4);
      else
        results.set(dmg, slope * (x + .5));
    }
    else
      results.set(dmg, 2 * height - slope * (x + .5));
  }

  return results;
}


function getWeaponMinDamage(weapon, upgradeLevel) {
  if (typeof weapon.minDamage === 'function')
    return weapon.minDamage(upgradeLevel);
  else
    return weapon.minDamage + upgradeLevel * weapon.minDamagePerLevel;
}

function getWeaponMaxDamage(weapon, upgradeLevel) {
  if (typeof weapon.maxDamage === 'function')
    return weapon.maxDamage(upgradeLevel);
  else
    return weapon.maxDamage + upgradeLevel * weapon.maxDamagePerLevel;
}

function getWeaponDamageDistribution(weapon, upgradeLevel, strength, isHuntress) {
  const weaponDamageDist = triangularIntDistribution(
    getWeaponMinDamage(weapon, upgradeLevel),
    getWeaponMaxDamage(weapon, upgradeLevel),
  );

  const strengthReq = weapon.strengthRequirement - upgradeLevel;

  if (!weapon.isRanged === !isHuntress && strength > strengthReq) {
    const totalDamageDist = new Map;
    const strBonusDamageDist = uniformIntDistribution(0, strength - strengthReq);

    for (const [strBonus, strBonusChance] of strBonusDamageDist) {
      for (const [damage, damageChance] of weaponDamageDist) {
        const totalDamage = damage + strBonus;
        totalDamageDist.set(totalDamage, (totalDamageDist.get(totalDamage) || 0) + damageChance * strBonusChance);
      }
    }

    return totalDamageDist;
  }

  return weaponDamageDist;
}


class SwarmOfFliesFarmingSimulator {
  constructor(damageDist) {
    this.damageDist_ = damageDist;
    this.resultCache_ = new Map;
  }

  /**
   * Simulates killing a Swarm of Flies, using the damage distribution used to
   * instantiate this object.
   */
  simulate(swarmHp = 80, generation = 0) {
    if (swarmHp <= 0)
      throw new Error(`Swarm is already dead at ${swarmHp} HP`);

    let swarmHpCache = this.resultCache_.get(swarmHp);
    if (swarmHpCache && swarmHpCache.has(generation))
      return swarmHpCache.get(generation);

    const result = { hits: 1, kills: 0, potions: 0, };

    for (const [damage, chance] of this.damageDist_) {
      if (damage <= 0)
        throw new Error(`Damage must be greater than 0; ${damage} was found.`);

      const swarmHpNew = swarmHp - damage;

      if (swarmHpNew <= 0) {
        // Swarm is killed
        const potionDropChance = 1 / (5 * (generation + 1));
        result.kills += chance;
        result.potions += chance * potionDropChance;
      }
      else if (swarmHpNew >= 2) {
        // Swarm splits in two
        const swarm2Hp = Math.floor(swarmHpNew / 2);
        const swarm1Hp = swarmHpNew - swarm2Hp;

        const swarm1Result = this.simulate(swarm1Hp, generation + 1);
        const swarm2Result = this.simulate(swarm2Hp, generation + 1);

        result.hits += chance * (swarm1Result.hits + swarm2Result.hits);
        result.kills += chance * (swarm1Result.kills + swarm2Result.kills);
        result.potions += chance * (swarm1Result.potions + swarm2Result.potions);
      }
      else {
        // Swarm will be killed on next attack
        const nextResult = this.simulate(swarmHpNew, generation);
        result.hits += chance * nextResult.hits;
        result.kills += chance * nextResult.kills;
        result.potions += chance * nextResult.potions;
      }
    }

    // Cache results
    this.resultCache_.set(swarmHp, swarmHpCache = this.resultCache_.get(swarmHp) || new Map);
    swarmHpCache.set(generation, result);

    return result;
  }
}


// Populate 'select weapon' field
document.getElementById('weapons').innerHTML = Object.entries(weapons).map(
  ([weaponId, weapon]) => {
    const text = weapon.name + (weapon.strengthRequirement ? ` [${weapon.strengthRequirement} STR]` : '');
    return `<option value="${weaponId}">${text}</option>`;
  }
).join('');


document.getElementById('swarm_form').addEventListener('change', event => {
  const strength = parseInt(document.getElementById('strength').value);
  const weaponId = document.getElementById('weapons').value;
  const isHuntress = document.getElementById('is_huntress').checked;

  const weapon = weapons[weaponId];

  const results = [];

  for (let weaponUpgrade = 0; weaponUpgrade <= 15; ++weaponUpgrade) {
    const damageDist = getWeaponDamageDistribution(weapon, weaponUpgrade, strength, isHuntress);
    const simulator = new SwarmOfFliesFarmingSimulator(damageDist);
    const simResult = simulator.simulate();

    const damageValues = Array.from(damageDist.keys());
    simResult.minDamage = Math.min(...damageValues);
    simResult.maxDamage = Math.max(...damageValues);
    simResult.weaponUpgrade = weaponUpgrade;

    results.push(simResult);
  }

  let tableHtml = '<table>'
    + '<tr>'
    + '<th>Weapon Upgrades</th>'
    + '<th>Damage</th>'
    + '<th># of Attacks</th>'
    + '<th># of Splits</th>'
    + '<th># of Potions Dropped</th>'
    + '<th># of Attacks to Get a Potion</th>'
    + '</tr>';

  for (const sim of results) {
    tableHtml += '<tr>'
      + '<td>' + sim.weaponUpgrade + '</td>'
      + '<td>' + sim.minDamage + '-' + sim.maxDamage + '</td>'
      + '<td>' + sim.hits.toFixed(2) + '</td>'
      + '<td>' + (sim.kills - 1).toFixed(2) + '</td>'
      + '<td>' + sim.potions.toFixed(3) + '</td>'
      + '<td>' + (sim.hits / sim.potions).toFixed(2) + '</td>'
      + '</tr>';
  }

  tableHtml += '</table>';

  document.getElementById('output_table').innerHTML = tableHtml;
});

document.getElementById('swarm_form').dispatchEvent(new Event('change'));

</script>