import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  const { gender } = req.body ?? {};
  if (!gender) return res.status(400).json({ error: 'gender required' });

  const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!);

  const { data: rows, error } = await supabase
    .from('items')
    .select('id, name, brand, image_url, ratings(rating, games)')
    .eq('active', true)
    .eq('gender', gender);

  if (error || !rows?.length) return res.status(200).json({ none: true });

  const items = rows.map(r => ({
    id: r.id,
    name: r.name,
    brand: r.brand,
    image_url: r.image_url,
    rating: r.ratings?.[0]?.rating ?? 1500,
    games: r.ratings?.[0]?.games ?? 0
  }));

  function pickPair() {
    const a = items[Math.floor(Math.random() * items.length)];
    let b = a;
    for (let i = 0; i < 50; i++) {
      const c = items[Math.floor(Math.random() * items.length)];
      if (c.id === a.id) continue;
      const dist = Math.abs(c.rating - a.rating);
      const bonus = 200 / Math.sqrt(1 + c.games);
      if (dist < 120 || Math.random() < Math.min(1, bonus / 400)) { b = c; break; }
      b = c;
    }
    const left = Math.random() < 0.5 ? a : b;
    const right = left.id === a.id ? b : a;
    return { left, right };
  }

  res.status(200).json(pickPair());
}
