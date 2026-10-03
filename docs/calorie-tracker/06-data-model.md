# 06. Модель данных

[← к оглавлению](README.md)

Общие правила для всех таблиц (кроме справочников схемы `ref`):
- первичный ключ `id uuid` (UUIDv7, может генерироваться клиентом);
- `family_id uuid` — изоляция данных семьи;
- `created_at`, `updated_at timestamptz`, `deleted_at timestamptz NULL` (мягкое удаление), `version int` (для оптимистичной блокировки);
- синхронизируемые таблицы пишутся в `change_log`.

## 1. Семья, участники, устройства

| Таблица | Ключевые поля |
|---|---|
| `family` | name, timezone, day_boundary_hour (по умолчанию 4), settings jsonb |
| `member` | name, aliases text[], sex, birth_date, height_cm, activity_level, goal (lose/maintain/gain), goal_rate, pregnancy_lactation, allergies text[], restrictions text[], disliked_dish_ids uuid[], visibility jsonb, has_account bool |
| `member_weight` | member_id, measured_at, weight_kg, source (manual/scale) |
| `account` | member_id, role (admin/adult/limited), status |
| `device` | account_id NULL (для общего), kind (phone/tablet/web), shared bool, platform, model, push_token, last_seen_at, revoked_at |
| `refresh_token` | device_id, token_hash, expires_at, rotated_from, revoked_at |
| `invite` | code_hash, role, member_id NULL, expires_at, redeemed_at, created_by |
| `audit_log` | actor_device_id, action, target, details jsonb, at |

## 2. Каталог

| Таблица | Ключевые поля |
|---|---|
| `dish` | name, type (homemade/store/outside/simple/composite), category, variant_group_id NULL, visual_description, unit_name NULL («котлета»), unit_mass_g, unit_mass_sd_g, density_g_ml, shelf_life_h, cook_time_min, food_item_id NULL (для простых), current_recipe_version_id NULL, barcode text[], place NULL, merged_into_id NULL, needs_review bool |
| `dish_synonym` | dish_id, text, lemma, source (user/learned), confirmed bool |
| `variant_group` | name («котлеты») |
| `dish_nutrients` | dish_id, recipe_version_id NULL, per100 jsonb {код_нутриента: значение}, completeness jsonb, source (recipe/label/analog), computed_at |
| `recipe_version` | dish_id, version, yield_g NULL (взвешенный), yield_factor, cooking_method, notes, created_by |
| `recipe_ingredient` | recipe_version_id, food_item_id NULL, dish_id NULL (ингредиент-полуфабрикат), mass_g, mass_type (gross/net), state (raw/cooked) |
| `composite_item` | composite_dish_id, dish_id, mass_g (для «моего обычного завтрака») |
| `dishware` | name, kind (plate/bowl/mug/glass/pot/pan), diameter_mm, depth_mm, volume_ml, tare_g, reference_photo_media_id, embedding vector(768) |
| `measure` | name («половник», «пиалка»), volume_ml, mass_g NULL, dish_id NULL (если мера специфична для блюда), member_id NULL |
| `member_habit` | member_id, dish_id, modifiers jsonb (молоко 2,5 %, без сахара), share (доля случаев) |

## 3. Медиа и эталоны

| Таблица | Ключевые поля |
|---|---|
| `media` | kind (photo/audio/thumb), storage_key, mime, bytes, width, height, duration_ms, taken_at, uploaded_by_device_id, status (pending/ready), retention_until |
| `reference_crop` | dish_id, media_id, region jsonb, embedding vector(768), embed_model_version, weight (1.0 / 0.7 карантин), is_negative bool, source_meal_item_id, created_at |

Индекс: HNSW по `reference_crop.embedding` (vector_cosine_ops), частичный по `deleted_at IS NULL`.

## 4. Приёмы пищи и партии

| Таблица | Ключевые поля |
|---|---|
| `meal_entry` | eaten_at, slot, source (photo/voice/text/barcode/repeat/plan), recognition_job_id NULL, created_by_device_id, shared_group_id NULL, status (draft/pending_recognition/saved), note |
| `meal_entry_member` | meal_entry_id, member_id, share |
| `meal_item` | meal_entry_id, member_id, dish_id, recipe_version_id NULL, batch_id NULL, mass_g, count NULL, mass_method (explicit/batch_weigh/unit_count/dishware/model/prior/measure), confidence_dish, confidence_mass, nutrients_snapshot jsonb (на 100 г), region jsonb NULL, media_id NULL, planned_meal_item_id NULL |
| `batch` | dish_id, recipe_version_id, cooked_at, cooked_by_member_id, modifications jsonb, initial_mass_g, mass_method (weighed/calculated), pot_dishware_id NULL, expires_at, closed_at, close_reason (eaten/discarded/expired) |
| `batch_event` | batch_id, kind (consume/weigh/discard), mass_g, meal_item_id NULL, at |

Остаток партии: `initial_mass_g − Σ consume − Σ discard` (вычисляется; материализуется в кэше).

## 5. Распознавание и обучение

| Таблица | Ключевые поля |
|---|---|
| `recognition_job` | input jsonb (media_ids, text), context jsonb, status (queued/running/completed/failed), decision, stages jsonb (результаты и время каждого этапа), prompt_version, model_name, embed_model_version, ranker_version, latency_ms, error |
| `recognition_hypothesis` | job_id, item_index, dish_id, features jsonb, score, rank, mass_estimates jsonb (по методам), mass_g, confidence_dish, confidence_mass |
| `recognition_feedback` | job_id, meal_item_id, kind (confirmed/dish_changed/mass_changed/item_added/item_removed/new_dish_created/variant_resolved), predicted_dish_id, actual_dish_id, predicted_mass_g, actual_mass_g, actual_mass_is_exact bool, weight, at |
| `context_prior` | member_id NULL (семейный), slot, weekday NULL, dish_id, value, updated_at |
| `portion_stat` | member_id, dish_id, median_g, iqr_g, cv, n, updated_at |
| `mass_calibration` | method, category, dishware_id NULL, k, sigma, n, updated_at |
| `ranker_model` | version, weights jsonb, metrics jsonb, active bool, trained_at |
| `confidence_calibration` | version, kind (dish/mass), points jsonb (изотоническая кривая), thresholds jsonb, active bool |
| `quality_metric_daily` | date, member_id NULL, top1, top3, auto_rate, correction_rate, mape_mass, median_log_time_s, n |
| `golden_set_item` | job_id, expected jsonb, added_at |

## 6. Нутриенты и нормы

| Таблица | Ключевые поля |
|---|---|
| `ref.food_source_item` | source (usda/ru_tables/ciqual/off/label), source_ref, name, raw jsonb |
| `ref.food_item` | name_ru, synonyms text[], category, edible_part, density_g_ml, nova_group NULL, retention_group, yield_factor_by_method jsonb |
| `ref.food_item_source_map` | food_item_id, source_item_id, match_score, verified bool |
| `ref.food_nutrient` | food_item_id, nutrient_code, value_per_100g, source, quality |
| `ref.nutrient` | code, name_ru, unit, importance_weight, is_limited (соль, сахар…) |
| `ref.retention_factor` | nutrient_code, retention_group, cooking_method, factor |
| `ref.nutrient_norm` | sex, age_from, age_to, condition (none/pregnancy/lactation), nutrient_code, rda, ul, unit, source |
| `member_norm` | member_id, nutrient_code, value, ul, is_manual, valid_from |
| `daily_intake` | member_id, date, nutrients jsonb, coverage jsonb, completeness_status (complete/incomplete/excluded), recomputed_at |
| `barcode_cache` | barcode, payload jsonb (Open Food Facts), fetched_at |

## 7. Аналитика, рекомендации, план

| Таблица | Ключевые поля |
|---|---|
| `recommendation` | member_id, period_start, problem jsonb, action jsonb, expected_effect jsonb, text, status (new/accepted/rejected/done), reject_reason, outcome jsonb |
| `week_template` | name, schedule jsonb (слоты, вне дома, совместные приёмы, время на готовку) |
| `meal_plan` | week_start, status (draft/approved/done), template_id, wishes text, constraints jsonb, explanation text, solver_stats jsonb, locked_by_device_id, locked_until |
| `planned_cooking` | plan_id, dish_id, recipe_version_id, day, responsible_member_id, total_mass_g, batch_id NULL (после готовки) |
| `planned_meal` | plan_id, day, slot, is_fixed bool, location (home/outside) |
| `planned_meal_item` | planned_meal_id, member_id, dish_id, planned_cooking_id NULL, mass_g |
| `plan_adherence` | planned_meal_id, member_id, status (done/done_substituted/replaced/skipped/unknown), match_score, energy_delta_kcal |
| `shopping_list_item` | plan_id, food_item_id NULL, dish_id NULL, name, quantity, unit, department, checked_by_member_id, checked_at |

## 8. Служебные

| Таблица | Ключевые поля |
|---|---|
| `change_log` | seq bigserial, family_id, entity, entity_id, op, changed_fields text[], author_device_id, at |
| `sync_op` | op_id (уникальный), device_id, applied_seq, result, at — для идемпотентности |
| `ai_usage` | job_id NULL, purpose, provider, model, input_tokens, output_tokens, audio_seconds, cost_usd, latency_ms, at |
| `notification` | member_id, type, payload jsonb, scheduled_at, sent_at, channel, dedup_key |
| `job_dead_letter` | queue, payload, error, attempts, at |

## 9. Ключевые связи (кратко)

```
family ─┬─ member ─┬─ account ── device
        │          ├─ member_weight, member_norm, portion_stat, member_habit
        │          └─ meal_entry_member ── meal_entry ── meal_item ─┬─ dish
        │                                                    │      ├─ batch ── batch_event
        │                                                    │      └─ planned_meal_item
        │                                                    └─ recognition_job ─┬─ recognition_hypothesis
        │                                                                        └─ recognition_feedback
        ├─ dish ─┬─ dish_synonym, reference_crop, dish_nutrients
        │        ├─ recipe_version ── recipe_ingredient ── ref.food_item
        │        └─ variant_group
        └─ meal_plan ─┬─ planned_cooking
                      ├─ planned_meal ── planned_meal_item
                      └─ shopping_list_item
```
