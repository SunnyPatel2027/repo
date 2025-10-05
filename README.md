# repo
import 'package:flutter/material.dart';
import 'package:loading_animation_widget/loading_animation_widget.dart';

class Loader {
  static void show(BuildContext context) {
    showDialog(
      context: context,
      barrierDismissible: false, // user cannot dismiss by tapping outside
      builder: (_) {
        return Center(
          child: LoadingAnimationWidget.threeRotatingDots(
            color: Colors.black,
            size: 60,
          ),
        );
      },
    );
  }

  static void hide(BuildContext context) {
    if (Navigator.canPop(context)) {
      Navigator.pop(context); // close the dialog
    }
  }
}
-----------------------
import 'package:dio/dio.dart';
import 'package:device_info_plus/device_info_plus.dart';
import 'dart:io';

class ApiClient {
  final Dio _dio = Dio(
    BaseOptions(
      baseUrl: 'http://esptiles.imperoserver.in/api/API/Product/',
      headers: {'Content-Type': 'application/json'},
    ),
  );

  Future<Response> post(String path, Map<String, dynamic> body) async {
    print("body = ${body}");

    return _dio.post(path, data: body);
  }

  Future<Map<String, String>> getDeviceInfo() async {
    final deviceInfo = DeviceInfoPlugin();

    if (Platform.isAndroid) {
      final androidInfo = await deviceInfo.androidInfo;
      return {
        "DeviceManufacturer": androidInfo.manufacturer,
        "DeviceModel": androidInfo.model,
      };
    } else if (Platform.isIOS) {
      final iosInfo = await deviceInfo.iosInfo;
      return {
        "DeviceManufacturer": "Apple",
        "DeviceModel": iosInfo.utsname.machine,
      };
    }
    return {
      "DeviceManufacturer": "Google",
      "DeviceModel": "Android SDK built for x86",
    };
  }
}
-----------------------


sealed class Result<T> {}

class Ok<T> extends Result<T> {
  final T data;
  Ok(this.data);
}

class Err<T> extends Result<T> {
  final String message;
  Err(this.message);
}
-----------------------
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_one/core/result.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_state.dart';
import 'package:practical_one/features/subcategory/data/catalog_repo.dart';
import 'package:practical_one/features/subcategory/domain/dashboard_model.dart';
import 'package:practical_one/features/subcategory/domain/product_data.dart';

class CatalogCubit extends Cubit<CatalogState> {
  final CatalogRepo repo;
  CatalogCubit({required this.repo})
    : super(
        CatalogState(
          selectedTab: 0,
          categoryData: [],
          subCategoryData: [],
          productData: [],
        ),
      );

  int pageCount = 1;
  bool isFetching = false;
  bool hasMore = true;

  /// Track product page count per subcategory
  final Map<int, int> _productPageTracker = {};

  /// Select tab
  onTapTab({required int tabIndex}) {
    emit(state.copyWith(selectedTab: tabIndex));
  }

  /// First load
  Future<void> loadFirst() async {
    emit(state.copyWith(isLoading: true));
    pageCount = 1;
    hasMore = true;
    isFetching = false;

    final res = await repo.fetchDaboardData(page: pageCount);
    if (res is Ok) {
      final dashboard = (res as Ok<DashboardModel>).data;
      emit(
        state.copyWith(
          isError: false,
          isLoading: false,
          categoryData: dashboard.resultData.categoryData,
          subCategoryData: dashboard.resultData.categoryData[0].subCategories,
          productData: dashboard.resultData.categoryData[0].subCategories
              .map((e) => e.product)
              .toList(),
        ),
      );
    } else if (res is Err) {
      emit(state.copyWith(isLoading: false, isError: true));
    }
  }

  /// Load more (pagination for subcategories)
  Future<void> loadSubCat() async {
    if (isFetching || !hasMore) return;
    isFetching = true;
    emit(state.copyWith(isSubCatLoading: true));

    final res = await repo.fetchDaboardData(
      page: pageCount + 1,
      catId: state.categoryData.isNotEmpty ? state.categoryData[0].id : 0,
    );

    if (res is Ok) {
      final dashboard = (res as Ok<DashboardModel>).data;
      final newCats = dashboard.resultData.categoryData[0].subCategories;

      if (newCats.isEmpty) {
        hasMore = false;
      } else {
        pageCount++;
        emit(
          state.copyWith(
            isError: false,
            subCategoryData: [...state.subCategoryData, ...newCats],
            productData: [
              ...state.productData,
              ...newCats.map((e) => e.product).toList(),
            ],
          ),
        );
      }
    } else if (res is Err) {
      emit(state.copyWith(isError: true));
    }

    emit(state.copyWith(isSubCatLoading: false));
    isFetching = false;
  }

  /// Load more products for a given subcategory index
  Future<void> loadProduct({required int subCatId, required int index}) async {
    if (isFetching) return;
    isFetching = true;

    emit(state.copyWith(isProductLoading: true));

    final int currentPage = _productPageTracker[subCatId] ?? 1;
    final res = await repo.fetchProductData(catId: subCatId, page: currentPage);

    if (res is Ok) {
      final data = (res as Ok<ProductDataModel>).data;

      if (data.result.isEmpty) {
        // No more products for this subcategory
        _productPageTracker[subCatId] = currentPage;
      } else {
        _productPageTracker[subCatId] = currentPage + 1;

        final List<List<ProductData>> updated = List.from(state.productData);
        if (index < updated.length) {
          updated[index].addAll(data.result);
        }

        emit(state.copyWith(productData: updated, isError: false));
      }
    } else if (res is Err) {
      emit(state.copyWith(isError: true));
    }

    emit(state.copyWith(isProductLoading: false));
    isFetching = false;
  }

  /// Refresh everything
  Future<void> refreshAll() async {
    pageCount = 1;
    hasMore = true;
    _productPageTracker.clear();
    await loadFirst();
  }
}
-------------------------

import 'package:practical_one/features/subcategory/domain/dashboard_model.dart';

class CatalogState {
  int selectedTab;
  bool isLoading;
  bool isSubCatLoading;
  bool isProductLoading;
  bool isError;

  List<CategoryData> categoryData;
  List<SubCategoriesData> subCategoryData;
  List<List<ProductData>> productData;

  CatalogState({
    required this.selectedTab,
    this.isLoading = false,
    this.isSubCatLoading = false,
    this.isProductLoading = false,
    this.isError = false,
    required this.categoryData,
    required this.subCategoryData,
    required this.productData,
  });

  CatalogState copyWith({
    int? selectedTab,
    bool? isLoading,
    bool? isSubCatLoading,
    bool? isProductLoading,
    bool? isError,
    List<CategoryData>? categoryData,
    List<SubCategoriesData>? subCategoryData,
    List<List<ProductData>>? productData,
  }) {
    return CatalogState(
      selectedTab: selectedTab ?? this.selectedTab,
      isLoading: isLoading ?? this.isLoading,
      isSubCatLoading: isSubCatLoading ?? this.isSubCatLoading,
      isProductLoading: isProductLoading ?? this.isProductLoading,
      isError: isError ?? this.isError,
      categoryData: categoryData ?? this.categoryData,
      subCategoryData: subCategoryData ?? this.subCategoryData,
      productData: productData ?? this.productData,
    );
  }
}
-----------------------------------

import 'package:practical_one/core/api_client.dart';
import 'package:practical_one/core/result.dart';
import 'package:practical_one/features/subcategory/domain/dashboard_model.dart';
import 'package:practical_one/features/subcategory/domain/product_data.dart';

class CatalogRepo {
  final ApiClient api;
  CatalogRepo({required this.api});

  Future<Result<DashboardModel>> fetchDaboardData({
    int page = 1,
    int catId = 0,
  }) async {
    try {
      final info = await api.getDeviceInfo();
      final res = await api.post("DashBoard", {
        ...info,
        "CategoryId": catId,
        "DeviceToken": "",
        "PageIndex": page,
      });

      print(
        "➡️ DashBoard API (page: $page, catId: $catId) => ${res.statusCode}",
      );

      if (res.statusCode == 200) {
        return Ok(DashboardModel.fromJson(res.data));
      } else {
        return Err("Unexpected response: ${res.statusCode}");
      }
    } catch (e) {
      print("❌ Error in fetchDaboardData: $e");
      return Err(e.toString());
    }
  }

  Future<Result<ProductDataModel>> fetchProductData({
    int page = 1,
    int catId = 0,
  }) async {
    try {
      final res = await api.post("ProductList", {
        "PageIndex": page,
        "SubCategoryId": catId,
      });

      print(
        "➡️ ProductList API (page: $page, subCatId: $catId) => ${res.statusCode}",
      );

      if (res.statusCode == 200) {
        return Ok(ProductDataModel.fromJson(res.data));
      } else {
        return Err("Unexpected response: ${res.statusCode}");
      }
    } catch (e) {
      print("❌ Error in fetchProductData: $e");
      return Err(e.toString());
    }
  }
}

----------------------------

class DashboardModel {
  DashboardModel({
    required this.status,
    required this.message,
    required this.resultData,
  });
  late final int status;
  late final String message;
  late final ResultData resultData;

  DashboardModel.fromJson(Map<String, dynamic> json) {
    status = json['Status'];
    message = json['Message'];
    resultData = ResultData.fromJson(json['Result']);
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Status'] = status;
    _data['Message'] = message;
    _data['Result'] = resultData.toJson();
    return _data;
  }
}

class ResultData {
  ResultData({required this.categoryData});
  late final List<CategoryData> categoryData;

  ResultData.fromJson(Map<String, dynamic> json) {
    categoryData = json == {}
        ? []
        : List.from(
            json['Category'],
          ).map((e) => CategoryData.fromJson(e)).toList();
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Category'] = categoryData.map((e) => e.toJson()).toList();
    return _data;
  }
}

class CategoryData {
  CategoryData({
    required this.id,
    required this.name,
    required this.isAuthorize,
    required this.update080819,
    required this.update130919,
    required this.subCategories,
  });
  late final int id;
  late final String name;
  late final int isAuthorize;
  late final int update080819;
  late final int update130919;
  late final List<SubCategoriesData> subCategories;

  CategoryData.fromJson(Map<String, dynamic> json) {
    id = json['Id'];
    name = json['Name'];
    isAuthorize = json['IsAuthorize'] ?? 1;
    update080819 = json['Update080819'];
    update130919 = json['Update130919'];
    subCategories = json['SubCategories'] == null
        ? []
        : List.from(
            json['SubCategories'],
          ).map((e) => SubCategoriesData.fromJson(e)).toList();
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Id'] = id;
    _data['Name'] = name;
    _data['IsAuthorize'] = isAuthorize;
    _data['Update080819'] = update080819;
    _data['Update130919'] = update130919;
    _data['SubCategories'] = subCategories.map((e) => e.toJson()).toList();
    return _data;
  }
}

class SubCategoriesData {
  SubCategoriesData({
    required this.id,
    required this.name,
    required this.product,
  });
  late final int id;
  late final String name;
  late final List<ProductData> product;

  SubCategoriesData.fromJson(Map<String, dynamic> json) {
    id = json['Id'];
    name = json['Name'];
    product = json['Product'] == null
        ? []
        : List.from(
            json['Product'],
          ).map((e) => ProductData.fromJson(e)).toList();
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Id'] = id;
    _data['Name'] = name;
    _data['Product'] = product.map((e) => e.toJson()).toList();
    return _data;
  }
}

class ProductData {
  ProductData({
    required this.name,
    required this.priceCode,
    required this.imageName,
    required this.id,
  });
  late final String name;
  late final String priceCode;
  late final String imageName;
  late final int id;

  ProductData.fromJson(Map<String, dynamic> json) {
    name = json['Name'];
    priceCode = json['PriceCode'];
    imageName = json['ImageName'];
    id = json['Id'];
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Name'] = name;
    _data['PriceCode'] = priceCode;
    _data['ImageName'] = imageName;
    _data['Id'] = id;
    return _data;
  }
}

import 'package:practical_one/features/subcategory/domain/dashboard_model.dart';

class ProductDataModel {
  ProductDataModel({
    required this.status,
    required this.message,
    required this.result,
  });
  late final int status;
  late final String message;
  late final List<ProductData> result;

  ProductDataModel.fromJson(Map<String, dynamic> json) {
    status = json['Status'];
    message = json['Message'];
    result = List.from(
      json['Result'],
    ).map((e) => ProductData.fromJson(e)).toList();
  }

  Map<String, dynamic> toJson() {
    final _data = <String, dynamic>{};
    _data['Status'] = status;
    _data['Message'] = message;
    _data['Result'] = result.map((e) => e.toJson()).toList();
    return _data;
  }
}

----------------------
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_one/core/widgets/loader.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_cubit.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_state.dart';
import 'package:practical_one/features/subcategory/data/catalog_repo.dart';
import 'package:practical_one/features/subcategory/presentation/sub_category_page.dart';

class CategoryScreen extends StatefulWidget {
  const CategoryScreen({super.key});

  @override
  State<CategoryScreen> createState() => _CatalogScreenState();
}

class _CatalogScreenState extends State<CategoryScreen> {
  final ScrollController subCatCtr = ScrollController();
  @override
  void initState() {
    super.initState();
    subCatCtr.addListener(() {
      if (subCatCtr.position.pixels >=
          subCatCtr.position.maxScrollExtent - 200) {
        // ✅ only one call per frame
        Future.microtask(() {
          context.read<CatalogCubit>().loadSubCat();
        });
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        backgroundColor: Colors.black,
        title: const Text("ESP Tiles", style: TextStyle(color: Colors.white)),
        centerTitle: true,
        actions: [
          IconButton(
            onPressed: () {},
            icon: const Icon(
              Icons.filter_alt_outlined,
              size: 25,
              color: Colors.white,
            ),
          ),
          IconButton(
            onPressed: () {},
            icon: const Icon(Icons.search, size: 25, color: Colors.white),
          ),
        ],
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(50),
          child: BlocConsumer<CatalogCubit, CatalogState>(
            buildWhen: (previous, current) =>
                previous.categoryData != current.categoryData ||
                previous.selectedTab != current.selectedTab,
            listener: (context, state) {
              if (state.isLoading) {
                Loader.show(context);
              } else {
                Loader.hide(context);
              }
            },
            builder: (context, state) {
              return state.isError
                  ? SizedBox()
                  : state.isLoading
                  ? SizedBox()
                  : SizedBox(
                      height: 50,
                      child: ListView.builder(
                        scrollDirection: Axis.horizontal,
                        shrinkWrap: true,
                        itemCount: state.categoryData.length,
                        itemBuilder: (context, index) {
                          final isSelected = state.selectedTab == index;
                          return GestureDetector(
                            onTap: () {
                              context.read<CatalogCubit>().onTapTab(
                                tabIndex: index,
                              );
                            },
                            child: Container(
                              padding: const EdgeInsets.symmetric(
                                horizontal: 16,
                                vertical: 10,
                              ),
                              margin: const EdgeInsets.symmetric(horizontal: 4),
                              decoration: BoxDecoration(
                                border: isSelected
                                    ? const Border(
                                        bottom: BorderSide(
                                          color: Colors.white,
                                          width: 2,
                                        ),
                                      )
                                    : null,
                              ),
                              child: Text(
                                state.categoryData[index].name,
                                style: TextStyle(
                                  color: isSelected
                                      ? Colors.white
                                      : Colors.grey[400],
                                  fontWeight: isSelected
                                      ? FontWeight.bold
                                      : FontWeight.normal,
                                  fontSize: 14,
                                ),
                              ),
                            ),
                          );
                        },
                      ),
                    );
            },
          ),
        ),
      ),
      body: Container(
        decoration: BoxDecoration(
          color: Colors.white,
          borderRadius: BorderRadius.only(
            topLeft: Radius.circular(10),
            topRight: Radius.circular(10),
          ),
        ),
        child: BlocBuilder<CatalogCubit, CatalogState>(
          buildWhen: (previous, current) =>
              previous.subCategoryData != current.subCategoryData ||
              previous.selectedTab != current.selectedTab,
          builder: (context, state) {
            return state.subCategoryData.isEmpty
                ? Container(
                    color: Colors.white,
                    child: Center(child: CircularProgressIndicator()),
                  )
                : state.selectedTab == 0 && state.subCategoryData.isNotEmpty
                ? SubCategoryPage(subCatCtr: subCatCtr)
                : Center(
                    child: Text(
                      "Selected: ${state.categoryData[state.selectedTab].name}",
                      style: const TextStyle(fontSize: 20),
                    ),
                  );
          },
        ),
      ),
    );
  }
}
-------------------------------

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_cubit.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_state.dart';
import 'package:practical_one/features/subcategory/domain/dashboard_model.dart';

class ProductWidget extends StatefulWidget {
  final List<List<ProductData>> product;

  final int index;
  final int subCatId;
  const ProductWidget({
    super.key,
    required this.product,
    required this.index,
    required this.subCatId,
  });

  @override
  State<ProductWidget> createState() => _ProductWidgetState();
}

class _ProductWidgetState extends State<ProductWidget> {
  final ScrollController productCtr = ScrollController();
  int productPageCount = 1;
  @override
  void initState() {
    super.initState();
    productCtr.addListener(() {
      if (productCtr.position.pixels >=
          productCtr.position.maxScrollExtent - 200) {
        // ✅ only one call per frame
        Future.microtask(() {
          context.read<CatalogCubit>().loadProduct(
            index: widget.index,
            subCatId: widget.subCatId,
          );
        });
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CatalogCubit, CatalogState>(
      buildWhen: (previous, current) =>
          previous.productData != current.productData ||
          previous.isProductLoading != current.isProductLoading,

      builder: (context, state) {
        return SizedBox(
          height: 150,
          child: ListView.builder(
            shrinkWrap: true,
            controller: productCtr,
            padding: EdgeInsets.zero,
            scrollDirection: Axis.horizontal,
            itemCount:
                widget.product[widget.index].length +
                (state.isProductLoading ? 1 : 0),
            itemBuilder: (context, i) {
              if (i == widget.product[widget.index].length) {
                return const Padding(
                  padding: EdgeInsets.all(8.0),
                  child: Center(child: CircularProgressIndicator()),
                );
              }
              final product = widget.product[widget.index][i];
              return Container(
                margin: const EdgeInsets.only(right: 20),
                width: 120,
                child: Column(
                  children: [
                    ClipRRect(
                      borderRadius: BorderRadius.circular(8),
                      child: Image.network(
                        product.imageName,
                        height: 100,
                        width: 120,
                        fit: BoxFit.cover,
                      ),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      product.name,
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(
                        fontSize: 11,
                        fontWeight: FontWeight.w400,
                        color: Colors.black,
                      ),
                    ),
                  ],
                ),
              );
            },
          ),
        );
      },
    );
  }
}
---------------------------------

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_cubit.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_state.dart';
import 'package:practical_one/features/subcategory/presentation/product_widget.dart';

class SubCategoryPage extends StatefulWidget {
  final ScrollController subCatCtr;
  const SubCategoryPage({super.key, required this.subCatCtr});

  @override
  State<SubCategoryPage> createState() => _SubCategoryPageState();
}

class _SubCategoryPageState extends State<SubCategoryPage> {
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CatalogCubit, CatalogState>(
      builder: (context, state) {
        if (state.isLoading && state.subCategoryData.isEmpty) {
          return const Center(child: CircularProgressIndicator());
        }
        if (state.isError) {
          return const Center(child: Text("Something went wrong"));
        }

        return Padding(
          padding: const EdgeInsets.all(10.0),
          child: ListView.builder(
            controller: widget.subCatCtr,
            itemCount:
                state.subCategoryData.length + (state.isSubCatLoading ? 1 : 0),
            itemBuilder: (context, index) {
              if (index == state.subCategoryData.length) {
                return const Padding(
                  padding: EdgeInsets.all(8.0),
                  child: Center(child: CircularProgressIndicator()),
                );
              }

              final subCat = state.subCategoryData[index];
              return Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    subCat.name,
                    style: const TextStyle(
                      color: Colors.black,
                      fontWeight: FontWeight.bold,
                      fontSize: 14,
                    ),
                  ),
                  const SizedBox(height: 8),
                  ProductWidget(
                    product: state.productData,
                    index: index,
                    subCatId: state.subCategoryData[index].id,
                  ),
                  const SizedBox(height: 20),
                ],
              );
            },
          ),
        );
      },
    );
  }
}
---------------------------------------

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_one/features/subcategory/bloc/catalog_cubit.dart';
import 'package:practical_one/features/subcategory/data/catalog_repo.dart';
import 'package:practical_one/features/subcategory/presentation/categoryPage.dart';
import 'core/api_client.dart';

void main() {
  final api = ApiClient();
  final repo = CatalogRepo(api: api);
  runApp(MyApp(repo: repo));
}

class MyApp extends StatelessWidget {
  final CatalogRepo repo;
  const MyApp({super.key, required this.repo});

  @override
  Widget build(BuildContext context) {
    return RepositoryProvider.value(
      value: repo,
      child: BlocProvider(
        create: (context) =>
            CatalogCubit(repo: context.read<CatalogRepo>())..loadFirst(),
        child: const MaterialApp(
          debugShowCheckedModeBanner: false,
          home: CategoryScreen(),
        ),
      ),
    );
  }
}
-------------------------
  flutter_bloc:
  dio:
  equatable:
  device_info_plus:
  loading_animation_widget: ^1.3.0
------------------------------------------------------------------------------------------------------------------------------------

import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter/material.dart';
import 'package:practical_two/features/test_strip/bloc/test_strip_state.dart';
import 'package:practical_two/features/test_strip/domain/test_strip_model.dart';

class TestStripCubit extends Cubit<TestStripState> {
  TestStripCubit() : super(const TestStripState()) {
    _loadParameters();
  }

  void _loadParameters() {
    final parameters = [
      TestStripParameter(
        name: "Total Hardness",
        unit: "ppm",
        values: [0, 110, 250, 500, 1000],
        colors: [
          Colors.purple[100]!,
          Colors.purple[300]!,
          Colors.purple,
          Colors.deepPurple,
          Colors.deepPurple[900]!,
        ],
        selectedColor: Colors.purple[100],
        selectedValue: 0,
      ),
      TestStripParameter(
        name: "Total Chlorine",
        unit: "ppm",
        values: [0, 1, 3, 5, 10],
        colors: [
          Colors.yellow[200]!,
          Colors.yellow,
          Colors.green[200]!,
          Colors.green,
          Colors.teal,
        ],
        selectedColor: Colors.yellow[200],
        selectedValue: 0,
      ),
      TestStripParameter(
        name: "Free Chlorine",
        unit: "ppm",
        values: [0, 1, 2, 3, 5],
        colors: [
          Colors.blue[100]!,
          Colors.lightBlue,
          Colors.blue,
          Colors.indigo,
          Colors.indigo[900]!,
        ],
        selectedColor: Colors.blue[100]!,
        selectedValue: 0,
      ),
      TestStripParameter(
        name: "pH",
        unit: "ppm",
        values: [6.2, 6.8, 7.2, 7.8, 8.4],
        colors: [
          Colors.red[200]!,
          Colors.red,
          Colors.deepOrange,
          Colors.orange,
          Colors.brown,
        ],
        selectedColor: Colors.red[200],
        selectedValue: 6.2,
      ),
      TestStripParameter(
        name: "Total Alkalinity",
        unit: "ppm",
        values: [0, 40, 120, 180, 180],
        colors: [
          Colors.green[100]!,
          Colors.green[300]!,
          Colors.green,
          Colors.teal,
          Colors.teal[900]!,
        ],
        selectedColor: Colors.green[100]!,
        selectedValue: 0,
      ),
      TestStripParameter(
        name: "Cyanuric Acid",
        unit: "ppm",
        values: [0, 50, 100, 150, 300],
        colors: [
          Colors.grey[200]!,
          Colors.blueGrey[200]!,
          Colors.blueGrey,
          Colors.blueGrey[700]!,
          Colors.black,
        ],
        selectedColor: Colors.grey[200]!,
        selectedValue: 0,
      ),
    ];

    emit(state.copyWith(parameters: parameters));
  }

  /// On tap color block
  void selectColor(int paramIndex, int colorIndex) {
    final updated = List<TestStripParameter>.from(state.parameters);
    final p = updated[paramIndex];
    updated[paramIndex] = p.copyWith(
      selectedValue: p.values[colorIndex],
      selectedColor: p.colors[colorIndex],
    );
    emit(state.copyWith(parameters: updated));
  }

  /// On textfield change
  void updateValue(int paramIndex, String input) {
    final updated = List<TestStripParameter>.from(state.parameters);
    final p = updated[paramIndex];
    if (input.isEmpty) return;
    final value = double.tryParse(input) ?? 0;
    final idx = p.getClosestIndex(value);
    updated[paramIndex] = p.copyWith(
      selectedValue: p.values[idx],
      selectedColor: p.colors[idx],
    );
    emit(state.copyWith(parameters: updated));
  }
}

import 'package:equatable/equatable.dart';
import 'package:practical_two/features/test_strip/domain/test_strip_model.dart';

class TestStripState extends Equatable {
  final List<TestStripParameter> parameters;
  final bool isLoading;

  const TestStripState({this.parameters = const [], this.isLoading = false});

  TestStripState copyWith({
    List<TestStripParameter>? parameters,
    bool? isLoading,
  }) {
    return TestStripState(
      parameters: parameters ?? this.parameters,
      isLoading: isLoading ?? this.isLoading,
    );
  }

  @override
  List<Object?> get props => [parameters, isLoading];
}


import 'package:flutter/material.dart';

class TestStripParameter {
  final String name;
  final String unit;
  final List<dynamic> values;
  final List<Color> colors;
  dynamic selectedValue;
  Color? selectedColor;

  TestStripParameter({
    required this.name,
    required this.unit,
    required this.values,
    required this.colors,
    this.selectedValue,
    this.selectedColor,
  });

  int getClosestIndex(double input) {
    double minDiff = double.infinity;
    int index = 0;
    for (int i = 0; i < values.length; i++) {
      double diff = (values[i] - input).abs();
      if (diff < minDiff) {
        minDiff = diff;
        index = i;
      }
    }
    return index;
  }

  TestStripParameter copyWith({dynamic selectedValue, Color? selectedColor}) {
    return TestStripParameter(
      name: name,
      unit: unit,
      values: values,
      colors: colors,
      selectedValue: selectedValue ?? this.selectedValue,
      selectedColor: selectedColor ?? this.selectedColor,
    );
  }
}


import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:practical_two/features/test_strip/bloc/test_strip_cubit.dart';
import 'package:practical_two/features/test_strip/bloc/test_strip_state.dart'
    show TestStripState;
import 'package:practical_two/features/test_strip/domain/test_strip_model.dart';

class TestStripScreen extends StatelessWidget {
  const TestStripScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => TestStripCubit(),
      child: Scaffold(
        appBar: AppBar(
          title: const Text(
            "Test Strip",
            style: TextStyle(fontWeight: FontWeight.bold),
          ),
          centerTitle: false,
        ),
        body: BlocBuilder<TestStripCubit, TestStripState>(
          builder: (context, state) {
            final cubit = context.read<TestStripCubit>();
            return Container(
              padding: const EdgeInsets.all(8),
              width: double.infinity,
              child: ListView.builder(
                itemCount: state.parameters.length,
                itemBuilder: (context, index) {
                  final param = state.parameters[index];
                  return ParameterRow(
                    parameter: param,
                    index: index,
                    onColorTap: (colorIndex) =>
                        cubit.selectColor(index, colorIndex),
                    onTextChange: (val) => cubit.updateValue(index, val),
                  );
                },
              ),
            );
          },
        ),
      ),
    );
  }
}

class ParameterRow extends StatefulWidget {
  final TestStripParameter parameter;
  final Function(int) onColorTap;
  final Function(String) onTextChange;
  final int index;
  const ParameterRow({
    super.key,
    required this.parameter,
    required this.onColorTap,
    required this.onTextChange,
    required this.index,
  });

  @override
  State<ParameterRow> createState() => _ParameterRowState();
}

class _ParameterRowState extends State<ParameterRow> {
  late TextEditingController _controller;
  late FocusNode _focusNode;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController(
      text: widget.parameter.selectedValue.toString(),
    );
    _focusNode = FocusNode();
  }

  @override
  Widget build(BuildContext context) {
    final param = widget.parameter;
    return Row(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Container(
          padding: EdgeInsets.symmetric(horizontal: 8),
          decoration: BoxDecoration(
            borderRadius: BorderRadius.only(
              topLeft: widget.index == 0
                  ? Radius.circular(8)
                  : Radius.circular(0),
              topRight: widget.index == 0
                  ? Radius.circular(8)
                  : Radius.circular(0),
              bottomLeft: widget.index == 5
                  ? Radius.circular(8)
                  : Radius.circular(0),
              bottomRight: widget.index == 5
                  ? Radius.circular(8)
                  : Radius.circular(0),
            ),
            border: Border(
              top: widget.index == 0
                  ? BorderSide(color: Colors.grey)
                  : BorderSide.none,
              left: BorderSide(color: Colors.grey),
              right: BorderSide(color: Colors.grey),
              bottom: widget.index == 5
                  ? BorderSide(color: Colors.grey)
                  : BorderSide.none,
            ),
          ),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const SizedBox(height: 45),
              Container(
                width: 10,
                height: 30,
                color: param.selectedColor ?? Colors.grey,
              ),
              const SizedBox(height: 50),
            ],
          ),
        ),
        const SizedBox(width: 10),
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    "${param.name} (${param.unit})",
                    style: const TextStyle(
                      fontWeight: FontWeight.bold,
                      color: Colors.grey,
                    ),
                  ),
                  SizedBox(
                    width: 80,
                    height: 35,
                    child: TextField(
                      controller: _controller,
                      focusNode: _focusNode,
                      keyboardType: TextInputType.number,
                      onTapOutside: (_) => _focusNode.unfocus(),
                      onChanged: widget.onTextChange,
                      decoration: InputDecoration(
                        border: OutlineInputBorder(
                          borderRadius: BorderRadius.circular(4),
                        ),
                        enabledBorder: OutlineInputBorder(
                          borderSide: const BorderSide(color: Colors.grey),
                          borderRadius: BorderRadius.circular(4),
                        ),
                        focusedBorder: OutlineInputBorder(
                          borderSide: const BorderSide(color: Colors.blue),
                          borderRadius: BorderRadius.circular(4),
                        ),
                        contentPadding: const EdgeInsets.symmetric(
                          vertical: 4,
                          horizontal: 8,
                        ),
                      ),
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 10),
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: List.generate(param.values.length, (i) {
                  return GestureDetector(
                    onTap: () => widget.onColorTap(i),
                    child: Column(
                      children: [
                        Container(
                          margin: const EdgeInsets.symmetric(horizontal: 4),
                          width: MediaQuery.of(context).size.width * 0.15,
                          height: 30,
                          decoration: BoxDecoration(
                            color: param.colors[i],
                            border: Border.all(
                              color: param.selectedValue == param.values[i]
                                  ? Colors.black
                                  : Colors.transparent,
                              width: 2,
                            ),
                          ),
                        ),
                        Text(param.values[i].toString()),
                      ],
                    ),
                  );
                }),
              ),
              const SizedBox(height: 25),
            ],
          ),
        ),
      ],
    );
  }
}


import 'package:flutter/material.dart';
import 'package:practical_two/features/test_strip/presentation/test_strip_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
      ),
      home: TestStripScreen(),
    );
  }
}

