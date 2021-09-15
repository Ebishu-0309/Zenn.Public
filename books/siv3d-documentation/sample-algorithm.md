---
title: "サンプル集 | アルゴリズムとデータ構造"
free: true
---

## kd 木

### 半径による探索と k 近傍法による探索

```cpp
# include <Siv3D.hpp>

struct Unit
{
	Circle circle;

	ColorF color;

	void draw() const
	{
		circle.draw(color);
	}
};

// Unit を KDTree で扱えるようにするためのアダプタ
struct UnitAdapter : KDTreeAdapter<Array<Unit>, Vec2>
{
	static const element_type* GetPointer(const point_type& point)
	{
		return point.getPointer();
	}

	static element_type GetElement(const dataset_type& dataset, size_t index, size_t dim)
	{
		return dataset[index].circle.center.elem(dim);
	}
};

void Main()
{
	// 4000 個の Unit を生成
	Array<Unit> units;
	for (size_t i = 0; i < 4000; ++i)
	{
		const Unit unit
		{
			.circle = Circle{ RandomVec2(Circle{100}), 0.25 },
			.color = RandomColorF(),
		};

		units << unit;
	}

	// kd-tree を構築
	KDTree<UnitAdapter> kdTree{ units };

	// 探索の種類（ラジオボタンのインデックス）
	size_t searchTypeIndex = 0;

	// radius search する際の探索距離
	double searchDistance = 4.0;

	// 2D カメラ
	Camera2D camera{ Vec2{ 0, 0 }, 24.0 };

	while (System::Update())
	{
		// 2D カメラの更新
		camera.update();

		// 画面内のユニットだけ処理するための基準の長方形
		const RectF viewRect = camera.getRegion();
		const RectF viewRectScaled = viewRect.scaledAt(viewRect.center(), 1.2);
		{
			const auto transformer = camera.createTransformer();

			const Vec2 cursorPos = Cursor::PosF();

			if (searchTypeIndex == 0) // radius search
			{
				Circle{ cursorPos, searchDistance }.draw(ColorF{ 1.0, 0.2 });

				// searchDistance 以内の距離にある Unit のインデックスを取得
				for (auto index : kdTree.radiusSearch(cursorPos, searchDistance))
				{
					Line{ cursorPos, units[index].circle.center }.draw(0.1);
				}
			}
			else // k-NN search
			{
				const size_t k = ((searchTypeIndex == 1) ? 1 : 5);

				// 最も近い k 個の Unit のインデックスを取得
				for (auto index : kdTree.knnSearch(k, cursorPos))
				{
					Line{ cursorPos, units[index].circle.center }.draw(0.1);
				}
			}

			// ユニットを描画
			for (const auto& unit : units)
			{
				// 描画負荷削減のため、画面内 (viewRectScaled) に無ければスキップ
				if (not unit.circle.center.intersects(viewRectScaled))
				{
					continue;
				}

				unit.draw();
			}
		}

		SimpleGUI::RadioButtons(searchTypeIndex, { U"radius", U"k-NN (k=1)", U"k-NN (k=5)" }, Vec2{ 20, 20 });
		SimpleGUI::Slider(U"searchDistance", searchDistance, 0.0, 20.0, Vec2{ 180, 20 }, 160, 120, (searchTypeIndex == 0));
		if (SimpleGUI::Button(U"Move units", Vec2{ 180, 60 }))
		{
			// Unit をランダムに移動
			for (auto& unit : units)
			{
				unit.circle.moveBy(RandomVec2(0.5));
			}

			// Unit の座標が更新されたので kd-tree を再構築
			kdTree.rebuildIndex();
		}

		camera.draw(Palette::Orange);
	}
}
```


### 長方形詰込み

```cpp
# include <Siv3D.hpp>

// ランダムな長方形の配列を作成
Array<Rect> GenerateRandomRects()
{
	Array<Rect> rects(Random(4, 32));

	for (auto& rect : rects)
	{
		const Point pos = RandomPoint(Rect{ Scene::Size() - Size{ 150, 150 } });
		rect.set(pos, Random(20, 150), Random(20, 150));
	}

	return rects;
}

void Main()
{
	Window::Resize(1280, 720);
	Scene::SetBackground(ColorF{ 0.99 });
	Array<Rect> input;
	Array<double> rotations;
	RectanglePack output;
	Point offset{ 0, 0 };
	Stopwatch stopwatch;

	while (System::Update())
	{
		if ((not stopwatch.isStarted()) || (1.8s < stopwatch))
		{
			input = GenerateRandomRects();
			rotations.resize(input.size());
			rotations.fill(0.0);

			// AllowFlip::Yes を指定すると 90° 回転を許可
			output = RectanglePacking::Pack(input, 1024, AllowFlip::Yes);

			for (size_t i = 0; i < input.size(); ++i)
			{
				if (input[i].w != output.rects[i].w)
				{
					rotations[i] = 270_deg;
				}
			}

			// 画面中央に表示するよう位置を調整
			offset = (Scene::Size() - output.size) / 2;
			for (auto& rect : output.rects)
			{
				rect.moveBy(offset);
			}

			stopwatch.restart();
		}

		// アニメーション
		const double k = Min(stopwatch.sF() * 10, 1.0);
		const double t = Math::Saturate(stopwatch.sF() - 0.2);
		const double e = EaseInOutExpo(t);

		Rect{ offset, output.size }.draw(ColorF{ 0.7, e });

		for (auto i : step(input.size()))
		{
			const RectF in = input[i].scaledAt(input[i].center(), k);
			const RectF out = output.rects[i];
			const Vec2 center = in.center().lerp(out.center(), e);
			const RectF rect{ Arg::center = center, in.size };

			rect.rotatedAt(rect.center(), Math::Lerp(0.0, rotations[i], e))
				.draw(HSV{ i * 25.0, 0.65, 0.9 })
				.drawFrame(2, 0, ColorF{ 0.25 });
		}
	}
}
```

### ボロノイ図

```cpp
# include <Siv3D.hpp>

void Main()
{
	Window::Resize(1280, 720);

	constexpr Size size{ 1280, 720 };
	constexpr Rect rect{ size };
	Subdivision2D subdiv{ rect };

	for (const PoissonDisk2D pd{ size, 40 }; 
		const auto& point : pd.getPoints())
	{
		if (rect.contains(point))
		{
			subdiv.addPoint(point);
		}
	}

	const Array<Polygon> facetPolygons = subdiv
		.calculateVoronoiFacets()
		.map([rect](const VoronoiFacet& f)
	{
		return Geometry2D::And(Polygon{ f.points }, rect).front();
	});

	while (System::Update())
	{
		for (auto [i, facetPolygon] : Indexed(facetPolygons))
		{
			facetPolygon
				.draw(HSV{ i * 25.0, 0.65, 0.8 })
				.drawFrame(2, ColorF{ 0.25 });
		}
	}
}
```


### ナビメッシュ

#### Polygon からナビメッシュを作成

```cpp
# include <Siv3D.hpp>

void Main()
{
	Window::Resize(1280, 720);
	Scene::SetBackground(ColorF{ 0.8, 0.9, 0.8 });

	Array<Vec2> points;
	Polygon polygon;
	LineString path;

	constexpr NavMeshConfig config{ .agentRadius = 20.0 };
	NavMesh navMesh;

	while (System::Update())
	{
		if (MouseL.down())
		{
			points << Cursor::Pos();
			polygon = Spline2D{ points }.calculateRoundBuffer(24, 8, 12);
			navMesh.build(polygon, config);
			path = navMesh.query(points.front(), points.back());
		}

		polygon.draw(ColorF{ 1.0 }).drawFrame(2, ColorF{ 0.7 });

		if (path)
		{
			path.draw(8, ColorF{ 0.1, 0.5, 0.9 });
			path.front().asCircle(12).draw(ColorF{ 1.0, 0.3, 0.0 });
			path.back().asCircle(12).draw(ColorF{ 1.0, 0.3, 0.0 });
		}
	}
}
```


#### 移動者の半径
大きい agent が細い道を通れないことを確認するサンプルです。

```cpp
# include <Siv3D.hpp>

void Main()
{
	Window::Resize(1280, 720);
	Scene::SetBackground(ColorF{ 0.1, 0.2, 0.2 });

	constexpr Size size{ 1280, 720 };
	constexpr Rect rect{ size };
	Subdivision2D subdiv{ rect };

	for (const PoissonDisk2D pd{ size, 40 };
		const auto & point : pd.getPoints())
	{
		if (rect.contains(point))
		{
			subdiv.addPoint(point);
		}
	}

	const Array<Polygon> facetPolygons = subdiv
		.calculateVoronoiFacets()
		.map([rect](const VoronoiFacet& f)
			{
				return Geometry2D::And(Polygon{ f.points }, rect).front();
			});

	Polygon mesh;
	NavMesh navMeshL, navMeshS;
	LineString pathL, pathS;
	constexpr double agentRadiusL = 30, agentRadiusS = 10;
	const Vec2 lStart{ 140, 120 };
	const Vec2 sStart{ 100, 200 };
	const Vec2 goal{ 1100, 300 };
	const Polygon goalDiamond = RectF{ Arg::center = goal, 48 }.rotated(45_deg).calculateRoundBuffer(3);

	while (System::Update())
	{
		for (const auto& facetPolygon : facetPolygons)
		{
			facetPolygon
				.draw(ColorF{ 0.3 })
				.drawFrame(2, ColorF{ 0.25 });
		}

		if (MouseL.pressed())
		{
			const Vec2 pos = Cursor::Pos();

			for (const auto& facetPolygon : facetPolygons)
			{
				if (facetPolygon.intersects(pos))
				{
					if (mesh.isEmpty())
					{
						mesh = facetPolygon;
					}
					else
					{
						mesh.append(facetPolygon);
						navMeshL.build(mesh, NavMeshConfig{ .agentRadius = agentRadiusL });
						navMeshS.build(mesh, NavMeshConfig{ .agentRadius = agentRadiusS });
						pathL = navMeshL.query(lStart, goal);
						pathS = navMeshS.query(sStart, goal);
					}

					break;
				}
			}
		}

		mesh.draw(ColorF{ 0.9, 0.8, 0.6 }).drawFrame(6, ColorF{ 0.5, 0.3, 0.0 });

		lStart.asCircle(agentRadiusL).draw(ColorF{ 0.2, 0.6, 0.5 });
		pathL.draw(6, ColorF{ 0.2, 0.6, 0.5 });

		sStart.asCircle(agentRadiusS).draw(ColorF{ 0.2, 0.3, 0.5 });
		pathS.draw(6, ColorF{ 0.2, 0.3, 0.5 });

		goalDiamond.draw(ColorF{ 0.9, 0.2, 0.0 });
	}
}
```


### 2D 計算幾何

#### Spline2D の曲率取得

```cpp
# include <Siv3D.hpp>

void Main()
{
	Window::Resize(1280, 720);
	Scene::SetBackground(ColorF{ 0.75 });

	Array<Vec2> points;
	Spline2D spline;

	Polygon polygon;
	Stopwatch stopwatch;
	SplineIndex si;

	while (System::Update())
	{
		// 制御点の追加
		if (MouseL.down())
		{
			points << Cursor::Pos();
			spline = Spline2D{ points, CloseRing::Yes };
			polygon = spline.calculateRoundBuffer(24);
			stopwatch.restart();
		}

		// 各区間の Bounding Rect の可視化
		for (size_t i = 0; i < spline.size(); ++i)
		{
			const ColorF color = Colormap01F(i / 18.0);
			spline.boundingRect(i)
				.draw(ColorF{ color, 0.1 })
				.drawFrame(1, 0, ColorF{ color, 0.5 });
		}

		// 点を追加してから 1 秒間は三角形分割を表示
		if (stopwatch.isRunning()
			&& (stopwatch < 1s))
		{
			polygon.drawWireframe(1, ColorF{ 0.25, (1.0 - stopwatch.sF()) });
			polygon.draw(ColorF{ 0.4, stopwatch.sF() });
		}
		else
		{
			polygon.draw(ColorF{ 0.4 });
			// 曲率に応じた色でスプラインを描画
			spline.draw(10, [&](SplineIndex si) { return Colormap01F(spline.curvature(si) * 24); });
		}

		// 制御点の表示
		for (const auto& point : points)
		{
			Circle{ point, 8 }.drawFrame(2, ColorF{ 0.8 });
		}

		// スプライン上を移動する物体
		if (spline)
		{
			si = spline.advanceWrap(si, Scene::DeltaTime() * 400);
			Circle{ spline.position(si), 20 }.draw(HSV{ 145, 0.9, 0.95 });
		}
	}
}
```

#### LineString の総距離と、スタート地点から指定した距離にある点

```cpp
# include <Siv3D.hpp>

void Main()
{
	Window::Resize(1280, 720);
	Scene::SetBackground(ColorF{ 0.6, 0.8, 0.7 });

	LineString points;
	Polygon polygon;

	double distanceFromOrigin = 0.0;
	double length = 0.0;

	while (System::Update())
	{
		if (MouseL.down())
		{
			points << Cursor::Pos();
			polygon = points.calculateRoundBuffer(20);
			length = points.calculateLength();
		}

		polygon.draw().drawFrame(2, ColorF{ 0.7 });
		points.draw(2, ColorF{ 0.75 });

		if (2 < points.size() && length)
		{
			distanceFromOrigin += (Scene::DeltaTime() * 800);

			if (length < distanceFromOrigin)
			{
				distanceFromOrigin = Math::Fmod(distanceFromOrigin, length);
			}

			const Vec2 position = points.calculatePointFromOrigin(distanceFromOrigin);
			position.asCircle(20).draw(ColorF{ 0.5 });
		}
	}
}
```

#### ハウスドルフ距離

```cpp
# include <Siv3D.hpp>

void Main()
{
	Scene::SetBackground(ColorF{ 1.0, 0.96, 0.92 });
	const Font font{ 40, Typeface::Heavy };

	const Polygon polygon = Shape2D::Star(240, Scene::Center());
	const LineString contour = polygon.outline();

	LineString contourClosed = contour;
	contourClosed << contour.front();

	// 計算の精度を高めるため LineString を細かくする
	const LineString base = contourClosed.densified(10.0);

	LineString lines;
	double distance = Math::Inf;

	while (System::Update())
	{
		contour.drawClosed(12, ColorF{ 0.7 });

		lines.draw(10, HSV{ 10, 1.0, 0.95 });

		if (MouseL.pressed())
		{
			lines << Cursor::Pos();

			// ハウスドルフ距離を計算
			distance = Geometry2D::HausdorffDistance(base, lines);
		}

		if (MouseR.pressed())
		{
			lines.clear();
			distance = Math::Inf;
		}

		if (IsFinite(distance))
		{
			font(U"{:.2f}"_fmt(distance)).draw(20, 20, ColorF{ 0.25 });
		}
	}
}
```

#### 図形の重なる領域

```cpp
# include <Siv3D.hpp>

void Main()
{
	const Polygon star = Shape2D::Star(200, Scene::Center());

	while (System::Update())
	{
		const Rect rect{ Arg::center = Cursor::Pos(), 300, 100 };

		star.draw(ColorF{ Palette::Yellow, 0.3 });

		rect.draw(ColorF{ 1.0, 0.8 });

		// star と rect の重なる領域を Polygon の配列で取得
		for (const auto& polygon : Geometry2D::And(star, rect))
		{
			polygon.draw(ColorF{ 1.0, 0.0, 0.0, 0.5 })
				.drawFrame(4, ColorF{ 1.0, 0.0, 0.0 });
		}
	}
}
```

#### 図形の引き算

```cpp
# include <Siv3D.hpp>

void Main()
{
	const Polygon star = Shape2D::Star(200, Scene::Center());

	while (System::Update())
	{
		const Rect rect{ Arg::center = Cursor::Pos(), 300, 100 };

		star.drawFrame(2, Palette::Yellow);

		rect.drawFrame(2, ColorF{ 1.0, 0.8 });

		// star から rect を引いた領域を Polygon の配列で取得
		for (const auto& polygon : Geometry2D::Subtract(star, rect))
		{
			polygon.draw(ColorF{ 1.0, 0.0, 0.0, 0.5 })
				.drawFrame(4, ColorF{ 1.0, 0.0, 0.0 });
		}
	}
}
```

#### 点群の凸包

```cpp
# include <Siv3D.hpp>

void Main()
{
	Array<Vec2> points;

	Polygon convexHull;

	while (System::Update())
	{
		if (MouseL.down())
		{
			// 点を追加
			points << Cursor::Pos();

			// 凸包を計算
			convexHull = Geometry2D::ConvexHull(points);
		}

		convexHull.draw(Palette::Skyblue);

		for (const auto& point : points)
		{
			Circle{ point, 5 }.draw(Palette::Seagreen);
		}
	}
}
```

#### Polygon の凸包

```cpp
# include <Siv3D.hpp>

void Main()
{
	const Polygon star = Shape2D::Star(200, Scene::Center(), 15_deg);

	// 凸包を計算
	const Polygon convexHull = star.computeConvexHull();

	while (System::Update())
	{
		convexHull.draw(Palette::Gray);

		star.draw(Palette::Yellow);
	}
}
```

#### Polygon の拡張

```cpp
# include <Siv3D.hpp>

void Main()
{
	Polygon polygon;

	while (System::Update())
	{
		const Rect rect{ Arg::center = Cursor::Pos(), 100 };

		if (MouseL.down())
		{
			// polygon に rect を追加する
			// ただし、polygon と rect がつながらない場合は失敗して false を返す
			polygon.append(rect);
		}

		polygon
			.draw(Palette::Skyblue)
			.drawWireframe(1, Palette::White);

		rect.drawFrame(1, 0, Palette::Skyblue);
	}
}
```

#### Polygon の太らせ、細らせ

```cpp
# include <Siv3D.hpp>

void Main()
{
	Scene::SetBackground(ColorF{ 0.6 });

	const Polygon starBase1 = Shape2D::Star(100, Scene::Center().movedBy(-160, 0));
	const Polygon starBase2 = Shape2D::Star(100, Scene::Center().movedBy(160, 0));

	Polygon star1 = starBase1;
	Polygon star2 = starBase2;

	double distance = 0;

	while (System::Update())
	{
		if (SimpleGUI::Slider(U"{:.1f}"_fmt(distance), distance, -30.0, 30.0, Vec2{ 20, 20 }))
		{
			// Polygon を太らせ / 細らせる
			star1 = starBase1.calculateRoundBuffer(distance);
			star2 = starBase2.calculateBuffer(distance);
		}

		star1.draw(Palette::Darkblue);
		star2.draw(Palette::Darkblue);

		starBase1.drawFrame(2, Palette::Yellow);
		starBase2.drawFrame(2, Palette::Yellow);
	}
}
```
