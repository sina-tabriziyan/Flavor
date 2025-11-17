val Context.globalDataStore: DataStore<Preferences> by preferencesDataStore(name = "global_prefs")
class GlobalCompanyManager(private val context: Context) : KoinComponent {
    private val COMPANY_ID_KEY = stringPreferencesKey("company_id")

    suspend fun setCompanyId(companyId: String) {
        context.globalDataStore.edit { prefs ->
            prefs[COMPANY_ID_KEY] = companyId
        }
    }

    val companyIdFlow: Flow<String?> = context.globalDataStore.data.map { prefs ->
        prefs[COMPANY_ID_KEY]
    }
}


import androidx.room.Room
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.*
import io.ktor.client.request.*
import io.ktor.http.*
import org.koin.core.qualifier.named
import org.koin.dsl.module

fun createCompanyScopedModule(companyId: String) = module {
    // Qualifier منحصربه‌فرد برای این scope
    val qualifier = named(companyId)

    // DataStore جدا برای این شرکت (نام فایل: company_{id}_prefs)
    single(qualifier) {
        val context: Context = get()
        context.createDataStore("company_${companyId}_prefs")
    }

    // Room Database جدا برای این شرکت
    single(qualifier) {
        val context: Context = get()
        Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "company_${companyId}.db"
        ).build()
    }

    // Ktor Client با Interceptor سفارشی
    single(qualifier) {
        val dataStore: DataStore<Preferences> = get(qualifier)
        HttpClient(CIO) {
            install(ContentNegotiation) {
                json()
            }
            install(HttpRequestBuilder) {
                // Interceptor: token را از DataStore بخوان و به header اضافه کن
                defaultRequest {
                    val token = dataStore.data.map { it[stringPreferencesKey("token")] }.firstOrNull() ?: ""
                    header(HttpHeaders.Authorization, "Bearer $token")
                    // سایر تنظیمات interceptor، مثل base URL بر اساس شرکت اگر نیاز باشد
                }
            }
        }
    }
}

// فرض کنید AppDatabase کلاس Room شما است
@Database(entities = [YourEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    // DAOs
}

// Extension برای ایجاد DataStore
fun Context.createDataStore(fileName: String): DataStore<Preferences> =
    PreferenceDataStoreFactory.create { dataStoreFile(fileName) }



    import androidx.compose.runtime.*
import androidx.navigation.NavHostController
import org.koin.androidx.compose.getKoin
import org.koin.core.component.KoinComponent
import org.koin.core.component.inject
import org.koin.core.qualifier.named
import org.koin.dsl.module
import kotlinx.coroutines.launch

@Composable
fun LoginScreen(navController: NavHostController, flavor: String) { // flavor: "kiosk" یا "restaurant"
    val koin = getKoin()
    val globalManager: GlobalCompanyManager by inject()
    val coroutineScope = rememberCoroutineScope()

    var selectedCompanyId by remember { mutableStateOf<String?>(null) }
    var companies by remember { mutableStateOf(listOf<String>()) } // لیست شرکت‌ها از سرور

    // دکمه دریافت اطلاعات سرور
    Button(onClick = {
        // فرض کنید API call برای گرفتن لیست شرکت‌ها (بدون scope، چون هنوز شرکت انتخاب نشده)
        // مثلاً با یک Ktor global ساده
        coroutineScope.launch {
            companies = fetchCompaniesFromServer() // implement کنید
        }
    }) {
        Text("دریافت اطلاعات سرور")
    }

    // انتخاب شرکت
    companies.forEach { companyId ->
        Button(onClick = {
            selectedCompanyId = companyId
            coroutineScope.launch {
                globalManager.setCompanyId(companyId)
                // ایجاد scope جدید
                val scopedModule = createCompanyScopedModule(companyId)
                koin.loadModules(listOf(scopedModule))
            }
        }) {
            Text("انتخاب شرکت: $companyId")
        }
    }

    // فیلدهای نام و پسورد
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    if (selectedCompanyId != null) {
        Button(onClick = {
            coroutineScope.launch {
                // گرفتن scope فعلی بر اساس companyId
                val qualifier = named(selectedCompanyId!!)
                val ktorClient: HttpClient = koin.get(qualifier)
                // لاگین با Ktor این scope (interceptor token را مدیریت می‌کند، اما اول بار token خالی است)
                val response = loginWithKtor(ktorClient, username, password)
                if (response.success) {
                    // ذخیره token در DataStore این scope
                    val dataStore: DataStore<Preferences> = koin.get(qualifier)
                    dataStore.edit { it[stringPreferencesKey("token")] = response.token }
                    // navigation به صفحه بعدی بر اساس flavor
                    navController.navigate(if (flavor == "kiosk") "kiosk_route" else "restaurant_route")
                }
            }
        }) {
            Text("لاگین")
        }
    }
}

// تابع نمونه برای لاگین
suspend fun loginWithKtor(client: HttpClient, username: String, password: String): LoginResponse {
    return client.post("https://your-server/login") {
        contentType(ContentType.Application.Json)
        setBody(LoginRequest(username, password))
    }.body()
}

// فرض کنید fetchCompaniesFromServer یک call ساده بدون scope است


class SomeRepository : KoinComponent {
    private val globalManager: GlobalCompanyManager by inject()
    
    fun getDataStore(): DataStore<Preferences> {
        val companyId = globalManager.companyIdFlow.value ?: throw Exception("No company selected")
        return getKoin().get(named(companyId))
    }
    
    // مشابه برای Ktor و Room
}



import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import io.ktor.client.*
import io.ktor.client.plugins.*
import io.ktor.client.request.*
import io.ktor.http.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.util.*
import kotlinx.coroutines.flow.first
import org.koin.core.qualifier.named
import org.koin.dsl.module
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

// Plugin سفارشی برای اینترسپتور token (مخصوص هر scope)
class CompanyAuthInterceptor(private val dataStore: DataStore<Preferences>) {
    private val TOKEN_KEY = stringPreferencesKey("token")

    suspend fun getToken(): String? = withContext(Dispatchers.IO) {
        dataStore.data.first()[TOKEN_KEY]
    }
}

// Install plugin در Ktor
fun HttpClientConfig<*>.installCompanyAuth(dataStore: DataStore<Preferences>) {
    install(HttpRequest) {
        val interceptor = CompanyAuthInterceptor(dataStore)
        interceptor { client ->
            val token = client.getToken()
            if (token != null) {
                header(HttpHeaders.Authorization, "Bearer $token")
            }
            // می‌تونی headerهای دیگه مثل Company-ID اضافه کنی
            header("X-Company-Id", "company_${dataStore}") // مثلاً برای شناسایی
        }
    }
}

fun createCompanyScopedModule(companyId: String) = module {
    val qualifier = named(companyId)

    // DataStore مخصوص این شرکت
    single<DataStore<Preferences>>(qualifier) {
        val context: android.content.Context = get()
        context.createDataStore("company_${companyId}_prefs")
    }

    // Room و بقیه... (همون کد قبلی)

    // Ktor Client با اینترسپتور مخصوص این scope
    single<HttpClient>(qualifier) {
        val dataStore: DataStore<Preferences> = get(qualifier)
        HttpClient(CIO) {
            install(ContentNegotiation) { json() }
            installCompanyAuth(dataStore) // اینترسپتور سفارشی که از همین DataStore token می‌خونه
            // تنظیمات دیگه مثل base URL اگر بر اساس شرکت فرق کنه
            defaultRequest {
                url("https://your-server.com/api") // یا dynamic بر اساس شرکت
            }
        }
    }
}




// در LoginScreen Compose (همون کد قبلی، اما با جزئیات بیشتر برای لاگین)
@Composable
fun LoginScreen(navController: NavHostController, flavor: String) {
    val koin = getKoin()
    val globalManager: GlobalCompanyManager by inject()
    val coroutineScope = rememberCoroutineScope()

    var selectedCompanyId by remember { mutableStateOf<String?>(null) }
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    // ... دکمه دریافت شرکت‌ها و انتخاب ...

    if (selectedCompanyId != null) {
        // فیلدهای نام و پسورد
        OutlinedTextField(value = username, onValueChange = { username = it }, label = { Text("نام") })
        OutlinedTextField(value = password, onValueChange = { password = it }, label = { Text("پسورد") })

        Button(onClick = {
            coroutineScope.launch {
                val qualifier = named(selectedCompanyId!!)
                val ktorClient: HttpClient = koin.get(qualifier) // این Ktor اینترسپتور مخصوص داره
                
                try {
                    // لاگین: اینترسپتور token خالی رو اضافه می‌کنه (اولین بار)
                    val response = ktorClient.post("/login") {
                        contentType(ContentType.Application.Json)
                        setBody(LoginRequest(username, password))
                    }.body<LoginResponse>()
                    
                    if (response.success) {
                        // ذخیره token در DataStore **همان scope** (پس اینترسپتور از این به بعد token رو می‌بینه)
                        val dataStore: DataStore<Preferences> = koin.get(qualifier)
                        dataStore.edit { prefs ->
                            prefs[stringPreferencesKey("token")] = response.token
                        }
                        
                        // حالا هر request بعدی با Ktor همین scope، token رو از اینترسپتور می‌گیره
                        navController.navigate(if (flavor == "kiosk") "kiosk_main" else "restaurant_main")
                    }
                } catch (e: Exception) {
                    // handle error
                }
            }
        }) {
            Text("لاگین")
        }
    }
}




class CompanyRepository : KoinComponent {
    private val globalManager: GlobalCompanyManager by inject()

    suspend fun fetchCompanyData() {
        val companyId = globalManager.companyIdFlow.first() ?: return
        val qualifier = named(companyId)
        val ktorClient: HttpClient = getKoin().get(qualifier) // اینترسپتور token رو اضافه می‌کنه
        
        val data = ktorClient.get("/companies/data").body<CompanyData>()
        // ...
    }
}




