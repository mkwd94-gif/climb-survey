import { createClient } from '@supabase/supabase-js';
function expected(ra:number, rb:number){ return 1/(1+Math.pow(10,(rb-ra)/400)); }

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  const { a_id, b_id, winner_id, session_id, ms_to_click, position_left, gender } = req.body ?? {};
  if (!a_id || !b_id || !gender) return res.status(400).json({ error: 'missing fields' });

  const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!);

  const { data: rA } = await supabase.from('ratings').select('*').eq('item_id', a_id).maybeSingle();
  const { data: rB } = await supabase.from('ratings').select('*').eq('item_id', b_id).maybeSingle();

  const A = rA ?? { item_id: a_id, rating: 1500, games: 0 };
  const B = rB ?? { item_id: b_id, rating: 1500, games: 0 };

  const Ea = expected(A.rating, B.rating);
  const Sa = winner_id === a_id ? 1 : (winner_id === b_id ? 0 : 0.5);
  const Sb = 1 - Sa;

  const kA = A.games < 20 ? 32 : (A.games < 100 ? 24 : 16);
  const kB = B.games < 20 ? 32 : (B.games < 100 ? 24 : 16);

  const newRa = A.rating + kA * (Sa - Ea);
  const newRb = B.rating + kB * (Sb - (1 - Ea));

  await supabase.from('votes').insert({
    a_id, b_id, winner_id, session_id, ms_to_click, position_left, a_rating: A.rating, b_rating: B.rating, gender
  });

  await supabase.from('ratings').upsert([
    { item_id: a_id, rating: newRa, games: A.games + 1 },
    { item_id: b_id, rating: newRb, games: B.games + 1 }
  ]);

  res.status(200).json({ ok: true });
}
